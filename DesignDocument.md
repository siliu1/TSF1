# TSF1.0 Design Document

## Overview
This document provides an overview of TSF1.0 framework support for Chromium, which allows IMEs to update content on a webpage through Windows Input Service. TSF1.0 framework also provides text intelligence features such as on-screen keyboard input suggestions, shape-writing in on-screen keyboard, customized on-screen keyboard UI for different types of edit control, that were not previously available in IMM32.


## Architecture Overview
Supporting TSF1.0 framework doesn’t require architectural changes to Chromium. At high level, TSF1.0 implementation lives in browser process and communicates with Windows Input Service via several COM interfaces. It communicates with renderer process through TextInputClient interface which is platform agnostic.  

TSFTextStore class implements ITextStoreACP, ITfContextOwnerCompositionSink, ITfTextEditSink and ITfKeyTraceEventSink interfaces provided by TSF1.0. When the browser process launches for the first time, TSFTextStore is created through the text store manager (TSFBridgeImpl).

TSF1.0 support contains three main components. Windows Input Service, TSF1.0 implementation in browser process, and TextInputClient which represents edit control in renderer process.

To work effectively and to provide text intelligence, Windows Input Service needs to know context of edit control which is currently in focus. Since renderer process can only communicate with browser process via IPC message mechanism, which is asynchronous, we need to maintain an Input Service’s view of the cached context for active edit control. The cache is stored in TSFTextStore. Windows Input Service directly works with the maintained cache synchronously. When browser process receives an IPC message from renderer process to update the cached text/selection/layout, we will run an algorithm to determine the difference between our cached context with updated context and notify Windows Input Service about the change. When Windows Input Service wants to modify text/selection/layout in edit control, TSFTextStore uses TextInputClient interface to modify the edit control asynchronously.

How many TSFTextStore objects in browser process? It would be ideal if we can create TSFTextStore for each edit control that is editable and provide full context to Windows Input Service. Due to limited knowledge of edit controls in browser process, TSFTextStore objects are created based on the predefined INPUT types and each input type shares the TSFTextStore object.

To show the relationship between each component, please read the following sections for a more detailed design.



## Detailed Design

### TSFTextStore creation and lifetime

When the browser process launches for the first time, it calls `BrowserMainRunnerImpl::Initialize` to initialize the main thread and other things like the main message loop, OLE etc. In this function, `ui::InitializeInputMethod` is called which is a static method that initializes the TSFBridgeImpl (a TLS implementation of TSFBridge). TSFBridgeImpl is created for a specific window handle and lives in the thread environment block of the UI thread. `TSFBridgeImpl::Initialize` method CoCreates the thread manager instance and then initializes ITfContexts for each TextInputType through `TSFBridgeImpl::InitializeDocumentMapInternal` API. The API also creates a document-to-TSFTextStore map based on TextInputType of the TextInputClient (implemented by RenderWidgetHostViewAura). It creates a TSFTextStore for each TextInputType, which can have following values:

* TEXT_INPUT_TYPE_NONE
* TEXT_INPUT_TYPE_TEXT
* TEXT_INPUT_TYPE_PASSWORD
* TEXT_INPUT_TYPE_SEARCH
* TEXT_INPUT_TYPE_EMAIL
* TEXT_INPUT_TYPE_NUMBER
* TEXT_INPUT_TYPE_TELEPHONE
* TEXT_INPUT_TYPE_URL 


### TSFTextStore post creation

Once TSFTextStore is successful registered with Windows Input Service, it is ready to receive function calls to modify content in edit control. Right after the edit control is in focus, TSFTextStore will hold a reference to the view (RenderWidgetHostViewAura), which implements TextInputClient interface. TSFTextStore uses TextInputClient interface to get updated context information of focused edit control. Under the hood, the context of edit control is stored in TextInputState struct in browser process. The struct lives in TextInputManager object which is owned by the view (RenderWidgetHostViewAura). When context is changed in edit control in renderer process, an IPC message will be sent to browser process to update the TextInputState struct.

When TSFTextStore receives function calls from Windows Input Service to insert text, change selection, complete composition, etc. TSFTextStore uses TextInputClient interface to do operations (delete a range, insert text, commit composition, etc) in renderer process. As soon as the IPC message reaches renderer process, InputMethodController is the object to do the work.


### Graphical explainer for the design

The communications between Windows Input Service(msctf.dll) and TSFTextStore happens on the UI thread of browser process and all communications are synchronous. The contract is that once the Input Service's view of context in edit control is locked (during OnLockGranted callback), the view of context should not be modified by components other than Windows Input Service. Hence, we come up with following design:

![msctf-tsftextstore](msctf-tsftextstore.PNG)

In the design, even though DOM tree is modified due to Javascript interuption during the lock, we won’t notify Windows Input Service about any change immediately. Instead, we calculate the difference and notify input service accordingly after the lock has been released. The TextInputState struct which is cached in browser process is being constantly updated from renderer process for any context change in edit control. We check the updated TextInputState struct with our cached context of edit control, if any difference is found meaning any text/selection/layout change is initialized from renderer process, then we will send notifications to input service.

The communication between TSFTextStore and renderer process are asynchronous. TSFTextStore is un-aware of any DOM operations, therefore all DOM related operations have to go through TextInputClient interface. We cache the Windows Input Service's view of edit control in TSFTextStore. We use the cached view to compare with TextInputState cache and calculate the text/selection that need to be changed in edit control and use TextInputClient to do the work after the lock has been released.

The diagram below shows the detailed design that messages go from Windows Input Service(msctf.dll) to renderer process through TSFTextStore. Function calls between msctf.dll and TSFTextStore are synchronous. Function calls between TSFTextStore and renderer process are asynchronous. Each rectangle indicates the text view of corresponding processes. For TSFTextStore, the left column is the Windows Input Service’s cached view of edit control, the right column is the TextInputState struct cache.

![msctf-tsftextstore-renderer](msctf-tsftextstore-renderer.PNG) 

The diagram below shows one complete cycle of IME composition with more complicated scenario. Given the nature of async communication, we should not work on the TextInputState cache until it’s updated.

![msctf-tsftextstore-renderer-composition](msctf-tsftextstore-renderer-composition.PNG)

In the design, we run the diffing algorithm after Windows Input Service completes the composition. However, the TextInputState struct in browser process hasn't been updated yet. We should not use the stale TextInputState struct to do the comparision. Instead, We run the diffing algorithm again after the TextInputState struct has been successfully updated and determines that there are changes in the edit control other than the change initialized by the Windows Input Service. We calculate the difference and notify Windows Input Service about the change initialized from renderer process.



## Links to relevant frameworks

* [Text Services framework](https://docs.microsoft.com/en-us/windows/desktop/TSF/text-services-framework)
* [TSF1.0 framework](https://docs.microsoft.com/en-us/windows/desktop/api/_tsf/)

## Appendix

* [Related Class Hierarchy](RelatedClassHierarchy.md)
* [Message Communications Between Components](MessageCommunications.md)
