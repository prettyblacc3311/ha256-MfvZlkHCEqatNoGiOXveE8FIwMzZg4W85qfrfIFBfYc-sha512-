// This content script executes various commands on behalf of the toolbar
// background page.

// Content
var Content = {
    widgetId: null,
    openFrameContainerId: null,
    openFrameId: null,
    openFrameIsModal: false,
    openFrameConfig: null,
    newToolbarFrame: null,
    highlightedText: '',
    initialized: false,

    logger : function logger(){
        var PREFIX = 'cS: ';
        try{
            var stackItems = [], stackItem, n;
            var e = new Error(),
                arr = e.stack.split('\n'),
                found = false,
                callStack = [],
                argsArr = [],
                textStr = '';
            stackItems = arr.slice(2);
            for(var i = 0; i<stackItems.length; i++){
                stackItem = stackItems[i];
                stackItem = stackItem.split(' ');
                for(var j = 0; j<stackItem.length; j++){
                    if(stackItem[j] == 'at' && !found){
                        n=stackItem[j+1];
                        callStack.push(stackItem[j+1]);
                        found=true;
                    }
                }
            }
            var name = arguments.callee.name || n;
            if(arguments){
                textStr = arguments[0];
                argsArr = Array.prototype.slice(arguments,1);
                argsArr.unshift(callStack);
                console.log.call(console,(PREFIX + name + ': %O : ' + textStr), argsArr);
            } else {
                console.log(PREFIX + name + ' invoked.');    
            }
        } catch (err){
            console.log(PREFIX + arguments.callee.name + ' ' +  err.stack);
            console.log(PREFIX + arguments.callee.name + ' ' +  err.message);    
        }

    },
    

    onDocumentReady : function onDocumentReady(callback){
        Content.logger();
        if( document && 
            document.body && 
            document.body.appendChild && 
            document.addEventListener && 
            document.removeEventListener){
                Content.logger('document ready');
                if (callback){
                    Content.logger('running callback');
                    callback();
                } else {
                    Content.logger('no callback');
                    return;
                }
        } else {
            Content.logger('not ready yet. Trying again.');
            setTimeout(Content.onDocumentReady, 100);
        }
    },

    initialize: function initialize(){
        Content.logger();
        var document = window.document;
        
        function tabComplete(){
            console.log('cS: sending TAB_COMPLETE');
            Messaging.send({ "name": 'TAB_COMPLETE' });
        }

        function onDocumentReadyDoInitialize(){
            Content.logger();
            Content.doInitialize();
            Content.logger('firing tab complete');
            tabComplete();
        }

        function removeVisibilityListenerCallBack(){
            Content.logger();
            if (window.handleVisibilityChange)
                document.removeEventListener("webkitvisibilitychange", window.handleVisibilityChange, false);
            onDocumentReadyDoInitialize();
        }

        if (!document.webkitHidden) {
            Content.onDocumentReady(onDocumentReadyDoInitialize);
        } else {
            Content.onDocumentReady(function addVisibilityListener(){
                Content.logger();
                document.addEventListener("webkitvisibilitychange", removeVisibilityListenerCallBack, false);
            });
            
        }
    },

    doInitialize: function doInitialize() {
        var self = this,
            mindspark = window.mindspark || {},
            ReserveSpaceForToolbar = mindspark.ReserveSpaceForToolbar;

        if (self.initialized) {
            console.log('cS: self.initialized');
            return;
        }

        //Create the toolbar iframe
        var createToolbarFrame = function createToolbarFrame(thisInstallDate) {
            
            var toolbarFrameByID = document.getElementById('iframeMyWebSearchToolbar');
            
            // check for any existing toolbars using iframeMyWebSearchToolbar
            if (toolbarFrameByID) {
                toolbarFrameByID.setAttribute('class', 'iframeMyWebSearchToolbar');
                toolbarFrameByID.setAttribute('installDate', 0);
            }

            var newFrame = document.createElement('iframe'),
                classNames = ['iframeMyWebSearchToolbar','Mindspark_-iframeMyWebSearchToolbar'];

            if (Common.showToolbarOnNewTab() && Common.hideTBSearch()){
                classNames.push('Mindspark_-iframePartOfPage');
            }
            var newTabOnly = Common.showToolbarOnNewTab() && !Common.showToolbarOnNormalPages();
            
            if(newTabOnly){
                console.log('cS: Toolbar is only shown on new tab');
                newFrame.style.zIndex = 500;    
            }


            newFrame.setAttribute('src', chrome.extension.getURL('toolbarUI.html'));
            newFrame.setAttribute('class', classNames.join(' '));
            newFrame.setAttribute('installDate', thisInstallDate);
            newFrame.style.display = 'none';
            newFrame.setAttribute('name', chrome.i18n.getMessage("@@extension_id"));
            
            // set handler to reposition tool bars after it is added to the DOM.
            newFrame.onload  = function newFrameLoadListener() {
                newFrame.style.opacity = 1;

                var toolbarFrameNodeList = document.getElementsByClassName('iframeMyWebSearchToolbar');
                var toolbarFrames = [];
                var i;
                for (i = 0; i < toolbarFrameNodeList.length; i++) {
                    toolbarFrames[i] = toolbarFrameNodeList[i];
                }

                toolbarFrames.sort(function (a, b) {
                    var installDate1 = a.getAttribute('installDate');
                    var installDate2 = b.getAttribute('installDate');
                    if (installDate1 < installDate2) {
                        return -1;
                    }
                    else if (installDate1 > installDate2) {
                        return 1;
                    }
                    return 0;
                });

                var offset = Common.isTBPartOfPage() ? 2 : 0;
                for (i = 0; i < toolbarFrames.length; i++) {
                    toolbarFrames[i].style.top = (offset + 30*i) + 'px';
                }

                // Ensure the toolbar is visible
                if (ReserveSpaceForToolbar){
                    ReserveSpaceForToolbar.verifyToolbarVisibility(
                        ReserveSpaceForToolbar.PAGE_EVENT.TOOLBAR_LOAD
                    );
                }
            };

            document.body.appendChild(newFrame);
            self.newToolbarFrame = newFrame;
        };

        var destroyToolbarFrame = function destroyToolbarFrame(installDate) {
            var toolbarFrame, i = 0,
                toolbarFrames = document.getElementsByClassName('iframeMyWebSearchToolbar');
            
            //remove desired tool bar and reposition.
            while (i < toolbarFrames.length) {
                toolbarFrame = toolbarFrames[i];
                
                if (installDate == toolbarFrame.getAttribute('installdate') ) {
                    document.body.removeChild(toolbarFrame);
                } else {
                    toolbarFrames[i].style.top = (30*(i)) + 'px';
                    i++;
                }
            }
        };

        // This is defined in reservespaceifenabled.js
        var toolbarData = window.toolbarData;

        if (toolbarData) {
            self.initialized = true;

            if (toolbarData.toolbarEnabled && ReserveSpaceForToolbar) {
                ReserveSpaceForToolbar.reserveSpace();
                createToolbarFrame(toolbarData.toolbarInfo.installTimestamp);
            }

            var injectionUrls = toolbarData.injectionUrls;

            if (injectionUrls) {
                var i = 0,
                    max = 0;

                for (i = 0, max = injectionUrls.length; i < max; i++) {
                    scriptInjector.injectFile(injectionUrls[i]);
                }
            }
        } else {
            console.log('cS: !toolbarData');
            return;
        }

        chrome.extension.onRequest.addListener(function onRequestListener(request, sender, sendResponse) {
            console.log('cs: onRequestListener(%O)', arguments);
            /*
             request should be of the form:
             {
             cmd: "REPLACE" | "ADD" | "REMOVE" | "REPAINT",
             containerId: "<element id>",
             html: "<the new html>"
             }
             REPLACE sets the inner html for the element with the given id
             ADD adds the given html to the page
             */

            var commandFound = true,
                asyncResponse = false,
                response = null;

            if (request.cmd == 'RESIZE') {
                var iframe = document.getElementById(self.openFrameId);
                if (iframe) {
                    var savedRequest;

                    if (window.popupDataById) {
                        savedRequest = window.popupDataById[request.containerId];
                    }

                    if (savedRequest) {
                        savedRequest.size = request.size;

                        Content.getWindowPositionInfo(savedRequest, function(positionInfo) {
                            iframe.style.width = positionInfo.width + 'px';
                            iframe.style.height = positionInfo.height + 'px';
                            iframe.style.top = positionInfo.top + 'px';

                            if (Common.isNotNull(positionInfo.left)) {
                                iframe.style.left = positionInfo.left + 'px';
                            } else if (Common.isNotNull(positionInfo.right)) {
                                iframe.style.right = positionInfo.right + 'px';
                            }
                        });
                    }
                }
            } else if (request.cmd == 'ADD') {
                var frameId = self.createFrameId(request);
                var renderWindow = function() {
                    Content.getWindowPositionInfo(request, function(positionInfo) {
                        var windowComponent = request.windowComponent || {},
                            width = positionInfo.width,
                            height = positionInfo.height,
                            top = positionInfo.top,
                            left = positionInfo.left,
                            dialogConfig = request.dialogConfig;

                        var transparent = windowComponent.transparent || request.transparent,
                            dialogClass = ( !transparent ? 'Mindspark_-dialog' : 'Mindspark_-dialog-transparent' );

                        if (dialogConfig) {
                            self.openFrameConfig = dialogConfig;
                        }

                        if (!window.popupDataById) {
                            window.popupDataById = {};
                        }

                        window.popupDataById[request.containerId] = request;

                        var iframe = document.getElementById(frameId);
                        if (iframe === null) {
                            iframe = document.createElement('iframe');
                            iframe.setAttribute('name', 'mindspark_' + chrome.i18n.getMessage("@@extension_id") + '_' + request.containerId);
                            iframe.setAttribute('class', dialogClass);
                            iframe.setAttribute('src', request.src);
                            iframe.setAttribute('id', frameId);
                            iframe.style.top = top + 'px';
                            iframe.style.width = width + 'px';
                            iframe.style.height = height + 'px';

                            if (Common.isNotNull(positionInfo.left)) {
                                iframe.style.left = positionInfo.left + 'px';
                            } else if (Common.isNotNull(positionInfo.right)) {
                                iframe.style.right = positionInfo.right + 'px';
                            }

                            if (request.border === false) {
                                iframe.style.border = 'none';
                            }

                            // If the response is sent in the load handler, the following error occurs sporadically:
                            // Attempting to use a disconnected port object [extensions/miscellaneous_bindings.js:187]
/*                          iframe.addEventListener("load", function() {
                                sendResponse({
                                    frameId: frameId,
                                    opened: true
                                });
                            }, false);*/

                            sendResponse({
                                frameId: frameId,
                                opened: true
                            });

                            document.body.appendChild(iframe);

                            self.openFrameId = frameId;
                            self.widgetId = request.containerId;
                            self.openFrameContainerId = request.containerId;

                            iframe.focus();
                        } else {
                            sendResponse({
                                frameId: frameId,
                                opened: false
                            });
                        }

                        if (dialogConfig && dialogConfig.modal) {
                            var modalOverlayId = self.createModalOverlayId(request.containerId),
                                modalOverlay = document.getElementById(modalOverlayId);

                            if (!modalOverlay) {
                                self.openFrameIsModal = true;

                                // Create Modal Overlay
                                modalOverlay = document.createElement('div');
                                modalOverlay.setAttribute('id', modalOverlayId);
                                modalOverlay.setAttribute('class', 'Mindspark_-modal-overlay');
                                modalOverlay.style.width = document.width + 'px';
                                modalOverlay.style.height = document.height + 'px';
                                document.body.appendChild(modalOverlay);
                            }
                        } else {
                            self.openFrameIsModal = false;
                        }
                    });
                };

                // Must occur first
                if (self.openFrameId != null && (!request.keepFrameOpen || self.openFrameId != frameId)) {
                    self.closeFrame(self.openFrameId, renderWindow);
                } else {
                    renderWindow();
                }

            } else if (request.cmd == 'REMOVE') {
                var closeFrameId;

                if (request.widgetWindow) {
                    closeFrameId = request.widgetWindow.frameId;
                } else {
                    closeFrameId = self.createFrameId(request);
                }

                response = self.closeFrame(closeFrameId);
            } else if (request.cmd == 'REPAINT') {
                // If this config does not exist, we are dealing with a legacy widget
                if (self.openFrameId != null && 
            (!self.openFrameConfig || !self.openFrameConfig.alwaysOnTop))
                {
                    self.closeFrame(self.openFrameId);
                }
            } else if (request.cmd == 'TURNON') {
                //Turn on toolbar
                createToolbarFrame(request.installTimestamp);
                ReserveSpaceForToolbar.reserveSpace();
            } else if (request.cmd == 'TURNOFF') {
                //Turn off toolbar
                destroyToolbarFrame(request.installTimestamp);
                ReserveSpaceForToolbar.destroyToolbar();
            } else if (request.cmd === 'CLOSE_FRAMES') {
                if (self.openFrameId != null) {
                    self.closeFrame(self.openFrameId);
                }
            } else if (request.cmd === 'ALERT') {
                alert(request.msg);
            } else if (request.cmd === 'GET_POSITIONING_INFO') {
                asyncResponse = true;

                Content.getWindowPositionInfo(request, function(positionInfo) {
                    sendResponse(positionInfo);
                });
            } else {
                commandFound = false;
            }

            if (!asyncResponse && sendResponse && response && commandFound) {
                sendResponse(response);
            }
        });

        document.documentElement.addEventListener('click', function(event) {
            if (self.openFrameId != null) {

                // If this config does not exist, we are dealing with a legacy widget
                if (!self.openFrameConfig) {
                    var openFrameElement = document.getElementById(self.openFrameId);
                    //Get the bounding box of the open frame
                    var top = openFrameElement.offsetTop;
                    var left = openFrameElement.offsetLeft;
                    var bottom = top + openFrameElement.offsetHeight;
                    var right = left + openFrameElement.offsetWidth;
                    //Get the click position
                    var x = event.clientX;
                    var y = event.clientY;

                    //Check to see if they clicking outside of the bounding box
                    if (!self.openFrameIsModal && (x < left || x > right || y < top || y > bottom)) {
                        self.closeFrame(self.openFrameId);
                    }
                } else {
                    if (self.openFrameConfig.menuLike) {
                        self.closeFrame(self.openFrameId);
                    }
                }
            }

            chrome.extension.sendRequest({ name: 'contentBodyClicked' });
        });

        //Listen to mouse and key ups to deal w/ text selection.
        document.body.addEventListener('mouseup', function(event) {
            self.textSelected();
        });

        document.body.addEventListener('keyup', function(event) {
            //If they just let go of the shiftkey, check for selection
            if (event.keyCode == 16) {
                self.textSelected();
            }
        });
    },

    textSelected: function() {
        var textSelection = window.getSelection().toString();
        if (textSelection && self.highlightedText !== textSelection) {
            self.highlightedText  = textSelection;
            chrome.extension.sendRequest({name: 'highlight', text: textSelection});
        }
    },

    createFrameId: function(request) {
        return "mindspark_" +
                ( request.namespaceId ? request.namespaceId : request.containerId ) +
                "_Frame";
    },

    createModalOverlayId: function(widgetId) {
        return 'mindspark_' + widgetId + '_ModalOverlay';
    },

    closeFrame: function(frameId, callback) {
        callback = callback || function() {};

        var elementToRemove = document.getElementById(frameId),
            removed = false;

        if (elementToRemove) {
            var parent = elementToRemove.parentNode;
            if (parent) {
                parent.removeChild(elementToRemove);

                // Remove the modal overlay if necessary
                if (this.openFrameIsModal) {
                    var modalOverlayId = this.createModalOverlayId(this.openFrameContainerId),
                        modalOverlay = document.getElementById(modalOverlayId);

                    document.body.removeChild(modalOverlay);
                }

                if (this.openFrameId == frameId) {
                    this.openFrameId = null;
                    this.openFrameContainerId = null;
                    this.openFrameConfig = null;
                }

                chrome.extension.sendRequest(
                    {
                        name: 'removeWindow',
                        frameId: frameId
                    },
                    callback
                );

                return;
            }
        }

        callback();
    },

    getWindowPositionInfo: function(message, callback) {
        // If there is no window component, this is a legacy widget and we must create it
        if (!message.windowComponent) {
            message.windowComponent = {
                type: 'dialog',
                anchor: 'button',
                width: message.width,
                height: message.height,
                path: message.src
            };

            callback(this.getAnchorPositionInfo(message, message.rectangle));
            return;
        }

        var self = this,
            windowComponent = message.windowComponent,
            type = windowComponent.type,
            anchor =  windowComponent.anchor;

        // Handle default anchors
        if (!anchor) {
            if (type === "dialog") {
                anchor = "button";
            } else if (type === "popup") {
                anchor = "screen";
            }

            windowComponent.anchor = anchor;
        }

        if (anchor === "button" || Common.isNull(anchor)) {
            if (typeof message.rectangle === "string") {
                var getRectangle = function(rectangle, _callback) {
                    chrome.extension.sendRequest(
                        {
                            name: 'getRectangle',
                            elementId: rectangle,
                            getAbsolutePosition: true
                        },
                        _callback
                    );
                };

                getRectangle(
                    message.rectangle,
                    function(rectangle) {
                        // The button could not be found which means the button is outside of the toolbar iframe.
                        // In this case, default to positioning relative to the widget button
                        if (rectangle !== -1) {
                            callback(self.getAnchorPositionInfo(message, rectangle));
                        } else {
                            getRectangle(
                                self.widgetId,
                                function(rectangle) {
                                    callback(self.getAnchorPositionInfo(message, rectangle));
                                }
                            );
                        }
                    }
                );
            } else if (typeof message.rectangle === "object") {
                callback(self.getAnchorPositionInfo(message, message.rectangle));
            }
        } else if (anchor === "searchBox") {
            chrome.extension.sendRequest(
                {
                    name: 'getRectangle',
                    elementId: "searchfor",
                    getAbsolutePosition: true
                },
                function(rectangle) {
                    callback(self.getAnchorPositionInfo(message, rectangle));
                }
            );
        } else {
            callback(self.getAnchorPositionInfo(message));
        }
    },

    getAnchorPositionInfo: function(message, anchorRectangle) {
        var self = this,
            windowComponent = message.windowComponent,
            size = message.size;

        if (!size) {
            if (message.width && message.height) {
                size = { width: message.width, height: message.height };
            } else {
                size = windowComponent;
            }
        }
        
        var domWindow = window,
            type = windowComponent.type,
            anchor = windowComponent.anchor,
            center = windowComponent.center,
            realSize = self.getRealSize(size, domWindow),
            width = realSize.width,
            height = realSize.height,
            toolbarIframe = this.newToolbarFrame,
            mindspark = window.mindspark || {ReserveSpaceForToolbar: {}},
            toolbarHeight = mindspark.ReserveSpaceForToolbar.toolbarHeight,
            toolbarOffsetTop = toolbarHeight;

        // Get iframe offSetHeight
        try {
            toolbarOffsetTop = getComputedStyle(toolbarIframe).getPropertyValue("top");
            toolbarOffsetTop = toolbarHeight + parseInt(toolbarOffsetTop.replace('px',''), 10);
        } catch(e) {}

        if (anchorRectangle) {
            anchorRectangle = self.getButtonAnchorRectangle(anchorRectangle, type, toolbarOffsetTop);
        } else {
            anchorRectangle = self.getAnchorRectangle(type, anchor, toolbarOffsetTop);
        }

        var x = anchorRectangle.left,
            y = anchorRectangle.top;

        var windowPositionInfo = {
            left: windowComponent.left,
            top: windowComponent.top,
            bottom: windowComponent.bottom,
            right: windowComponent.right,
            width: width,
            height: height
        };

        // Declare function and automatically execute
        var formatPositionInfo = (function() {
            // Ensure the window component's position info are integers
            var format = function() {
                _.map(windowPositionInfo, function(value, key, list) {
                    if (_.isString(value)) {
                        if (_.isEmpty(value)) {
                            list[key] = null;
                        } else {
                            list[key] = parseInt(value, 10);
                        }
                    } else if (_.isNumber(value)) {
                        list[key] = parseInt(value, 10);
                    }
                });
            };

            format();

            return format;
        }());

        if (center === "horizontal" || center === "both") {
            x += ( anchorRectangle.width - width ) / 2;

            // Left should override right in case they are both specified
            if (Common.isNotNull(windowPositionInfo.left)) {
                x += windowPositionInfo.left;
            } else if (Common.isNotNull(windowPositionInfo.right)) {
                x -= windowPositionInfo.right;
            }
        } else {
            // Left should override right in case they are both specified
            if (Common.isNotNull(windowPositionInfo.left)) {
                x += windowPositionInfo.left;
            } else if (Common.isNotNull(windowPositionInfo.right)) {
                // Take advantage of the "right" css property if this is a dialog
                if (anchor !== "button" && type === "dialog") {
                    x = null;
                } else {
                    x += anchorRectangle.width - width - windowPositionInfo.right;
                }
            }
        }

        if (center === "vertical" || center === "both") {
            y += ( anchorRectangle.height - height ) / 2;

            // Top should override bottom in case they are both specified
            if (Common.isNotNull(windowPositionInfo.top)) {
                y += windowPositionInfo.top;
            } else if (Common.isNotNull(windowPositionInfo.bottom)) {
                y -= windowPositionInfo.bottom;
            }
        } else {
            // Top should override bottom in case they are both specified
            if (Common.isNotNull(windowPositionInfo.top)) {
                y += windowPositionInfo.top;
            } else if (anchor !== "button") {
                if (Common.isNotNull(windowPositionInfo.bottom)) {
                    y += anchorRectangle.height - height - windowPositionInfo.bottom;
                }
            }
        }

        // Display button-anchored windows in a user-friendly manner
        if (anchor === "button") {
            var boundingRectangle;

            // Define bounding rectangle. Dialogs are bound by the browser tab, popups by the screen
            if (type === "dialog") {
                boundingRectangle = self.getAnchorRectangle(type, "tab", toolbarOffsetTop);
            } else {
                boundingRectangle = self.getAnchorRectangle(type, "screen", toolbarOffsetTop);
            }

            var newCoordinates = Mindspark_.adapterUtil.getBoundedWindowPosition({
                "x": x,
                "y": y,
                "windowPositionInfo": windowPositionInfo,
                "anchorRectangle": anchorRectangle,
                "boundingRectangle": boundingRectangle,
                "enforceAxisY": type === "popup"
            });

            x = newCoordinates.x;
            y = newCoordinates.y;
        }

        windowPositionInfo = _.extend(windowPositionInfo, {
            left: x,
            top: y
        });

        formatPositionInfo();

        return windowPositionInfo;
    },

    getButtonAnchorRectangle: function(rectangle, type, toolbarOffsetTop) {
        var domWindow = window,
            top = rectangle.top,
            left = rectangle.left;

        if (!rectangle.isOffset) {
            top += toolbarOffsetTop;

            rectangle.isOffset = true;
        }

        if (type === "popup") {
            top += domWindow.screenY + domWindow.outerHeight - domWindow.innerHeight;
            left += domWindow.screenX;
        }

        return _.extend(rectangle, {
            top: top,
            left: left
        });
    },

    getAnchorRectangle: function(type, anchor, toolbarOffsetTop) {
        var domWindow = window,
            domScreen = domWindow.screen,
            rectangle = {},
            domWindowInnerHeight = domWindow.innerHeight - toolbarOffsetTop;

        // Dialogs are always iframes rendered in the DOM
        if (type === "dialog") {
            rectangle = {
                width: domWindow.innerWidth,
                height: domWindowInnerHeight,
                left: 0,
                top: toolbarOffsetTop
            };
        } else {
            if (anchor === "tab") {
                rectangle = {
                    width: domWindow.innerWidth,
                    height: domWindowInnerHeight,
                    left: domWindow.screenX,
                    top: domWindow.screenY + domWindow.outerHeight - domWindowInnerHeight
                };
            } else if (anchor === "browser") {
                rectangle = {
                    width: domWindow.outerWidth,
                    height: domWindow.outerHeight,
                    left: domWindow.screenX,
                    top: domWindow.screenY
                };
            } else if (anchor === "screen") {
                rectangle = {
                    width: domScreen.availWidth,
                    height: domScreen.availHeight,
                    left: domScreen.availLeft,
                    top: domScreen.availTop
                };
            }
        }

        return rectangle;
    },

    getRealSize: function(size, contentWindow) {
        // Handle width and height percentages
        var height = size.height,
            width = size.width,
            documentInnerWidth = contentWindow.innerWidth,
            documentInnerHeight = contentWindow.innerHeight;
        if (Common.isPercentage(width)) {
            var widthPercentage = parseInt(width, 10) / 100;

            width = widthPercentage * documentInnerWidth;
        } else {
            // Coerce to number
            width *= 1;
        }

        if (Common.isPercentage(height)) {
            var heightPercentage = parseInt(height, 10) / 100;

            height = heightPercentage * documentInnerHeight;
        } else {
            // Coerce to number
            height *= 1;
        }
        return {width: width, height: height};
    }
};

(function(window) {
    Content.initialize();
}(window));
