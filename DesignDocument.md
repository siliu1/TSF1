# TSF1.0 Design Document


## Summary

This document provides an overview of the proposed changes to Chromium to support TSF1.0, enabling IMEs to update content on a webpage through the Windows Input Service. TSF1.0 also provides text intelligence features such as on-screen keyboard input suggestions, shape-writing in on-screen keyboard, customized on-screen keyboard UI for different types of edit control that are not available in IMM32. A partial implementation of TSF1.0, disabled by default, exists in Chromium today.  This document describes a way to implement full support for TSF1.0 in Chromium, building on the existing implementation.


### Platforms

Windows Desktop.


### Code affected

The changes impact content.dll and ui_base_ime.dll.


## Overview

Windows Input Service facilitates communication between input methods (handwriting recognition, speech recognition, shape-writing, etc), and editing applications that want text input (code editors, rich text editing applications like Microsoft Word, or web browsers). Each editing application has a model for representing the content it edits, and to avoid the need for each input method to code against N different editor document models, Windows Input Service defines an intermediate language to facilitate communication between the editing applications and the input methods: a shared plain-text buffer of content.

The shared buffer is updated cooperatively with the help of Windows Input Service. When an input method delivers input to an editing application, it does so by requesting that Windows Input Service update the shared buffer, and Windows Input Service can in response notify the editing app how the buffer is being modified so that the editing application can update its document model. Selection state, layout state highlighting state and composition state are used to describe the state in the buffer. Selection state reflects current selection in the document model. When input method requests a selection range, Windows Input Service gets up-to-date selection range from the document model and reports the selection range to the input method. Highlighting of text in the buffer can also be requested by input method so it can indicate what text is being composed in response to phonetic input. When input method requests a highlight over the range to show the candidate window, Windows Input Service queries for the bounding rectangle coordinates of focused editing control and editing application is responsible to draw highlight on the range requested by Windows Input Service. After the user finishes composing and commits final composition text, Windows Input Service updates the shared buffer with committed text and changes composition state to notify editing application that composition has been completed. Editing app is responsible for clearing highlighted range in the document model and replace the highlighted text with committed text.

### Constraint and Principle

TSF1 requires a synchronous response to queries about the state of the shared buffer, yet the input thread of the browser process must remain responsive. To acheive both requirements, no blocking cross-process calls or waiting for script to yield so the state of the DOM can be read will be allowed; instead all queries must be fulfilled from cached state maintained in the browser process that mirrors (with some imperceptible delay) the contents of the DOM in the renderer process.

Since both input methods and editing applications are trying to update the same editable content, conflicts can occur. To resolve conflicts, the order of operations as seen by the DOM will be considered authoritative.

After resolving the conflict, the shared buffer may not reflect the input delivered by Windows Input Service, an algorithm must be used to determine and notify the input method about the additional change made to the shared buffer.

### Key assumption

The view of the editable content provided to Windows Input Service should be aligned with what the user sees on the screen. Transient state of the DOM that the user cannot see is irrelevant to the editing process and the view provided of the editable content to Windows Text Service.

We are free to reuse TSF objects that describe the content of an editable region of an editing app as there is no known data cached using the identity of these objects, i.e. all that is necessary to facilitate a great editing experience can be computed from the contents of the DOM and populated into an appropriate TSF object as focus shifts from one editable element to another.


## Architecture

TSF1.0 support involves three key components: Windows Input Service, a TSF1.0 component in the browser process, and TextInputClient representing edit control in a renderer process. The TSF1.0 component communicates with renderer processes via the TextInputClient interface, and uses several COM interfaces to communicate with the Windows Input Service. When the browser process launches for the first time, TSFTextStore, which implements ITextStoreACP, ITfContextOwnerCompositionSink, ITfTextEditSink and ITfKeyTraceEventSink interfaces provided by Windows Input Service, is created by TSFBridgeImpl, which is the manager of TSFTextStore, for each input type. Once TSFTextStore is successfully registered with Windows Input Service, it is ready to receive function calls to modify content in edit control. Windows Input Service uses TSFTextStore to query for editing context in Blink in order to do operations such as insert text, change selection, update composition, etc. 

When the browser process launches for the first time, it calls `BrowserMainRunnerImpl::Initialize` to initialize the main thread and other things like the main message loop, OLE etc. In this function, `ui::InitializeInputMethod` is called which is a static method that initializes the TSFBridgeImpl (a TLS implementation of TSFBridge). TSFBridgeImpl is created for a specific window handle and lives in the thread environment block of the UI thread. `TSFBridgeImpl::Initialize` method CoCreates the thread manager instance which is unique for current HWND. Thread manager then initializes ITfContexts for each TextInputType through `TSFBridgeImpl::InitializeDocumentMapInternal` API. The API also creates a 1-to-1 relationship of ITfDocumentMgr-to-TSFTextStore map based on TextInputType of the TextInputClient (implemented by RenderWidgetHostViewAura). It creates a TSFTextStore for each TextInputType, which can have following values:

* TEXT_INPUT_TYPE_NONE
* TEXT_INPUT_TYPE_TEXT
* TEXT_INPUT_TYPE_PASSWORD
* TEXT_INPUT_TYPE_SEARCH
* TEXT_INPUT_TYPE_EMAIL
* TEXT_INPUT_TYPE_NUMBER
* TEXT_INPUT_TYPE_TELEPHONE
* TEXT_INPUT_TYPE_URL 

We have created thread manager instance, ITfDocumentMgr assotiated with TSFTextStore for each TextInputType. Once TSFTextStore is successfully registered with the Windows Input Service, it is ready to receive function calls to modify content in edit control. Right after the edit control is in focus, TSFTextStore will hold a reference to the view (RenderWidgetHostViewAura), which implements TextInputClient interface. Since TSFTextStore doesn't cache any editing context of focused edit control, Windows Input Service gets empty text and selection of (0,0) after querying for the editing context from TSFTextStore. TSFTextStore uses TextInputClient interface to do operations (delete a range, insert text, commit composition, etc) in renderer process. As soon as the mojo message reaches renderer process, InputMethodController is the final object to do the work.

Here is a graphical view of the architecture between each components:

![components](components.PNG)

For more detailed architecture overview, please see [communication between components](MessageCommunications.md) and [related class hierarchy](RelatedClassHierarchy.md).


## Issue with current TSF1.0 implementation and proposed mitigation

The Windows Input Service needs to know the context of the focused edit control. However, the current implementation of TSFTextStore doesn't provide this context except during IME (active) composition. The focused edit control is always empty with selection of (0,0) from the perspective of the Windows Input Service. New IME composition is always started at (0,0) because TSFTextStore always reports an empty buffer in the edit control. TSFTextStore also sends text change notification to Windows Input Service to clear the buffer after the IME composition has been committed. This issue blocks Windows Input Service from providing full functionality of text intelligence features such as on-screen keyboard suggestions. As a result, for example, a Korean IME cannot perform reconversion on already-committed text.

To address this, TSFTextStore needs to be made aware of the editing context of the focused edit control. The editing context of the focused edit control is stored in a TextInputState struct in the browser process. The struct lives in TextInputManager which is owned by the view (RenderWidgerHostViewAura). When the editing context is changed in edit control, a mojo message is sent to the browser process to update the TextInputState struct to stay in sync with the focused edit control. TSFTextStore obtains the editing context of focused edit control from TextInputState struct and notifies the Windows Input Service with this context.


## Details

After enabling TSFTextStore to be aware of the existing editing context, we also need to maintain the Input Service’s view of the editing context for focused edit control because a renderer process can only communicate with the browser process via mojo communication asynchronously. The cache is stored in TSFTextStore. Windows Input Service directly modifies the maintained cache because TSFTextStore communicates with Windows Input Service synchronously. Windows Input Service uses TSFTextStore to do editing operations in focused edit control, and TSFTextStore uses TextInputClient interface to modify the edit control asynchronously. A DOM operation may trigger additional javascript calls which may modify the DOM tree as well. Any context change in the focused edit control will trigger a mojo communication between the renderer process and browser process to update TextInputState struct. When TextInputState is updated from renderer process, we run an algorithm to determine the difference between our cached editing context with updated editing context in edit control. The difference will be the additional change not triggered by Windows Input Service, therefore, we notify Windows Input Service about the change.


### Graphical explainer for the design

The communications between Windows Input Service (msctf.dll) and TSFTextStore happens on the UI thread of the browser process and all communications are synchronous. The contract is that once the Input Service's view of context in edit control is locked (during OnLockGranted callback), the view of context should not be modified by components other than msctf.dll. One complete locking period is considered as one entire edit session. Hence, we chose the following approach:

![msctf-tsftextstore](msctf-tsftextstore.PNG)

Even when the DOM tree is modified due to Javascript interuption during the lock, we won’t notify Windows Input Service about any change immediately to prevent reentrancy. Instead, we calculate the difference and notify input service accordingly after the lock has been released. The TextInputState struct which is cached in browser process is being constantly updated from renderer process for any context change in edit control. We check the updated TextInputState struct with our cached context of edit control, if any difference is found meaning any text/selection/layout change is initialized from renderer process, then we will send notifications to input service.

Windows Input Service (msctf.dll) can do editing operations during each edit session. Since communication between TSFTextStore and renderer process is asynchronous and communication between msctf.dll and TSFTextStore are synchronous, we update msctf.dll's view of edit control right after TSFTextStore receives function calls from msctf.dll. After the lock has been released, we now use the cached view to compare with TextInputState struct and calculate the text/selection that need to be changed in edit control and use TextInputClient to do the work.

The diagram below shows the detailed design that messages go from msctf.dll to renderer process through TSFTextStore. Function calls between msctf.dll and TSFTextStore are synchronous. Function calls between TSFTextStore and renderer process are asynchronous. Each rectangle indicates the text view of corresponding processes. For TSFTextStore, the left column is msctf.dll’s cached view of edit control, the right column is the TextInputState struct.

![msctf-tsftextstore-renderer](msctf-tsftextstore-renderer.PNG) 

The diagram below shows one complete cycle of IME composition with more complicated scenario. Given the nature of async communication, we should not work on the TextInputState cache until it’s updated.

![msctf-tsftextstore-renderer-composition](msctf-tsftextstore-renderer-composition.PNG)

In the design, we run the diffing algorithm after Windows Input Service completes the composition. However, the TextInputState struct in the browser process hasn't been updated yet. We should not use the stale TextInputState struct to do the comparision. Instead, We run the diffing algorithm again after the TextInputState struct has been successfully updated and determines that there are changes in the edit control other than the change initialized by the Windows Input Service. We calculate the difference and notify Windows Input Service about the change initialized from renderer process.


### Race conditions and solutions

First race condition happens within TSFTextStore. When we run diffing algorithm after each edit session, there may be a case where the msctf.dll's cached view is different from TextInputState struct. For example, initial text buffer of the focused edit control is "12" and selection is (1,1). msctf.dll tries to insert "a" during one edit session and the cached view in TSFTextStore is "1a2". However, when "a" is inserted into the DOM tree, Javascript changes the buffer in edit control to "3b4". When we run the algorithm to determine the change after the lock is released, the cached view's "1a2" is different from TextInputState's "3b4". In this case, the editing context in TextInputState always takes precedence. We notify msctf.dll about context change with TextInputState's context.

A second race condition happens due to the asynchronous mojo communication between browser process and renderer process. There may be a case where the current edit session has ended but the TextInputState hasn't been updated yet. In this case we shouldn't use stale TextInputState struct to run diffing algorithm. We should wait and run the algorithm until the TextInputState is updated. The solution is also illustrated in the graphical explainer above.


### Considerations for TSFTextStore Reuse

For TSF1.0 Support, it would be ideal if we can create TSFTextStore for each edit control that is editable and provide full editing context to Windows Input Service. Text intelligence features are relying on the context in the focused edit control. TSFTextStore is used to report the context to Windows Input Service. If we have a 1-to-1 relationship between TSFTextStore and editable elements, then the context in TSFTextStore is preserved when focus changes to another editable element.

However, browser process has no knowledge about DOM element concept and TSFTextStore lives inside the browser process. TSFTextStore cannot be created for each edit control in current Chromium architecture. Therefore, TSFTextStore objects are created based on the predefined INPUT types and each input type shares the TSFTextStore object.

In order to achieve 1 to 1 relationship between TSFTextStore and edit control, we would need to modify existing Chromium architecture to expose edit control to browser process. Such a change will impact all platforms. By following the existing architecture, we re-use TSFTextStore when focus switches from first edit control to second edit control with same predefined input type. the cached context in TSFTextStore is changed to the context of the second edit control. When the focus switches back to the first edit control, the context cached in TSFTextStore is changed back to the context of the first edit control.

No text intelligence features are affected because we re-populate the context of the edit control everytime the edit control is in focus. The downside is that we need to notify msctf.dll about the context change everytime focus switches from one edit control to another.


## Performance

We expect the TSF1.0 support will have similar performance as IMM32 support on Windows because both implementations use the same TextInputClient interface to achieve editing outcomes. We will add telemetry logs to TSF1.0 implementations to compare the performance with IMM32.


## Testing plan

Unit tests are written for TSFTextStore, including edge-case handling. We also do manual testing by installing popular IMEs on all Windows platforms and test it on popular editing sites.


## Implementation and Shipping Plan

There is an existing feature flag for TSF1.0 support. We will follow the standard Chrome guidelines to enable by default.


## Links to relevant frameworks

* [Text Services framework](https://docs.microsoft.com/en-us/windows/desktop/TSF/text-services-framework)
* [TSF1.0 framework](https://docs.microsoft.com/en-us/windows/desktop/api/_tsf/)

## Appendix

* [Related Class Hierarchy](RelatedClassHierarchy.md)
* [Message Communications Between Components](MessageCommunications.md)
