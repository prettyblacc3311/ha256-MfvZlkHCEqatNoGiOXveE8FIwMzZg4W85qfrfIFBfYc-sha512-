var api = (function() {
	"use strict";

	var API_VERSION = "1.4",
		adapterUtil = Mindspark_.adapterUtil,
		_ = Mindspark_.underscore;

    var isSupportedProtocol = function(tab){
        return !/^chrome-extension:/.test(tab.url);
    };

	var isToolbarEnabled = function(tab, callback) {
		if (tab && window.Toolbar) {
			Toolbar.isToolbarEnabled(tab.url, tab, callback);
		} else {
			callback(false);
		}
	};

    var handleNewTabURL = function(url){
        return /chrome:\/\/newtab\//.test(url) ? "" : url;
    };

    var createURLDetails = function createURLDetails(url){
        var urlObj = Mindspark_HttpURL(url),
            details = {
                fullURL: url,
                scheme: urlObj.getScheme(),
                domain: urlObj.getDomain(),
                port: urlObj.getPort(),
                path: urlObj.getPath(),
                fragmentId: urlObj.getFragmentId(),
                queryString: urlObj.getQueryString(),
                params: urlObj.getParamsObject()
            };
        return details;
    };

	var getWidgetWindowTabId = function(windowId, callback) {
		chromeUtils.tabs.getSelected(
			{ windowId: windowId, windowType: 'popup' },
			function(tab) {
				callback(tab ? tab.id : null);
			}
		);
	};

	var getWidgetWindow = function(windowId, tab) {
		var widgetWindow;
		if (windowId) {
			widgetWindow = widgetWindowManager.getWindow(windowId);
		} else if (tab) {
			widgetWindow = widgetWindowManager.getWindowByTab(tab);
		}
		return widgetWindow;
	};

	var init = function(widgetInstance) {
		var config = widgetInstance.config,
			widgetId = config.id,
			background = config.background,
			button = config.button;

		var listeners = {
			"GetTabInfo": function getTabInfoListener(message, sender, sendResponse) {
				var tabId = message.body;

				var tabCallback = function getTabInfoTabCallback(tab) {
					if (Common.isNull(tab)) {
						widgetInstance.getSelectedTab(function getTabInfoGetSelectedTabCallback(tab) {
							if (tab) {
								isToolbarEnabled(tab, function getTabInfoGetSelectedTabIsToolbarEnabledCallback(enabled) {
									sendResponse({
										url:             handleNewTabURL(tab.url),
                                        urlDetails:      createURLDetails(tab.url),
										active:          true,
										tabId:           tab.id,
										isToolbarVisible:enabled
									});
								});
							}
						});
					} else {
						isToolbarEnabled(tab, function tabCallbackIsToolbarEnabledCalback(enabled) {
							if (tab.active) {
								chrome.windows.get(tab.windowId, function(window) {
									var active = window && window.focused;

									sendResponse({
										url: handleNewTabURL(tab.url),
                                        urlDetails: createURLDetails(tab.url),
										active: active,
										tabId: tab.id,
										isToolbarVisible: enabled
									});
								});
							} else {
								sendResponse({
									url: handleNewTabURL(tab.url),
                                    urlDetails: createURLDetails(tab.url),
									active: false,
									tabId: tab.id,
									isToolbarVisible: enabled
								});
							}
						});
					}
				};

				if (Common.isEmpty(tabId)) {
					chrome.tabs.getCurrent(tabCallback);
				} else {
					chrome.tabs.get(tabId, tabCallback);
				}
			},

			"GetWidgetConfigData": function(message, sender, sendResponse) {
				sendResponse(Common.extend({}, config.data, message.body));
			},
			"GetToolbarSettings": function(message, sender, sendResponse) {
                var toolbarDataStr = Global.retrieve('dlpToolbarData'),
                    toolbarData = JSON.parse(toolbarDataStr),
				    localStorageUrl = Common.localStorageCommunicationUrl,
                    installSuccessURL = window.config.installSuccess,
                    newTabBubbleURL = localStorageUrl.replace(/blank\.jhtml/, 'chromeInstruct.jhtml?tabView=bubble'),
                    newDownloadURL = localStorageUrl.replace(/blank\.jhtml/, 'chromeInstruct.jhtml'),
                    newTabInstructURL = localStorageUrl.replace(/blank\.jhtml/, 'chromeInstruct.jhtml?tabView=instruct'),
                    newTabSuccessURL = localStorageUrl.replace(/blank\.jhtml/, 'chromeInstruct.jhtml?tabView=success'),
                    settings = {
                        toolbarId: Global.getToolbarId(),
                        partnerId: Global.getPartnerId(),
                        partnerSubId: Global.getPartnerSubId(),
                        installDate: Global.getInstallDate(),
                        toolbarVersion: window.config.version,
                        apiVersion: API_VERSION,
                        dlp: {
                            newTabBubbleURL: newTabBubbleURL,
                            newDownloadURL: newDownloadURL,
                            newTabInstructURL: newTabInstructURL,
                            newTabSuccessURL: newTabSuccessURL
                        }
                    };
                for (var key in toolbarData){
                    if (toolbarData.hasOwnProperty(key)){
                        settings.dlp[key] = toolbarData[key];
                    }
                }
				var keys = message.body;
				var returnSettings = function() {
					sendResponse(Common.extend({}, settings, keys));
				};
				if (Common.isNull(keys) || keys.indexOf("toolbarLanguage") != -1) {
					chrome.i18n.getAcceptLanguages(function(languages) {
						settings.toolbarLanguage = languages[0];
						returnSettings();
					});
				} else {
					returnSettings();
				}
			},

			"GetStoredData": function(message, sender, sendResponse) {
				var keys = message.body,
					data = {},
					prefix = widgetId + '_',
					copyFromLocalStorage = function(key) {
						var value = localStorage.getItem(prefix + key);
						data[key] = value === '' ? null : value;
					},
					i,
					length,
					prefixLength;

				if (Common.isNull(keys)) {
					// load all data with the prefix of this widget
					length = localStorage.length;
					prefixLength = prefix.length;
					for (i = 0; i < length; i++) {
						var key = localStorage.key(i);
						if (key.indexOf(prefix) == 0) {
							copyFromLocalStorage(key.substring(prefixLength));
						}
					}
				}
				else if (typeof(keys) == "string") {
					copyFromLocalStorage(keys);
				}
				else if (Array.isArray(keys)) {
					keys.forEach(copyFromLocalStorage);
				}
				else {
					throw "Invalid keys type: " + typeof(keys);
				}
				sendResponse(data);
			},

			"SetStoredData": (function() {
				var	BYTES_PER_KB = 1024,
					BYTES_PER_CHAR = 2,
					MAX_VALUE_KB = 64,
					MAX_VALUE_BYTES = MAX_VALUE_KB * BYTES_PER_KB,
					MAX_VALUE_CHARS = MAX_VALUE_BYTES / BYTES_PER_CHAR - 1;

				return function(message, sender, sendResponse) {
					var data = message.body,
						processedData = {},
						maxSizeLimitExceeded = false;

					// Pre-processing must be done to ensure no values exceed the maximum size
					Common.each(data, function(prop) {
						if (maxSizeLimitExceeded) {
							return;
						}

						var key = widgetId + '_' + prop.key,
							value = prop.value === null ? '' : prop.value;

						if (!_.isNull(value)) {
							if (!_.isUndefined(value)) {
								if (!_.isString(value)) {
									value = JSON.stringify(value);
								}

								if (value.length <= MAX_VALUE_CHARS) {
									processedData[key] = value;
								} else {
									maxSizeLimitExceeded = true;
								}
							} else {
								// undefined should be handled as a null value
								value = null;
							}
						}
					});

					if (!maxSizeLimitExceeded) {
						Common.each(processedData, function(prop) {
							localStorage.setItem(prop.key, prop.value);
						});

						sendResponse();
					} else {
						sendResponse("TOOBIG");
					}
				};
			}()),

			"AjaxRequest": function(message, sender, sendResponse) {
				adapterUtil.sendAjaxRequest(message.body, sendResponse);
			},

			"NavigateNewTab": function(message, sender, sendResponse) {
                //now supports ability to navigate to the New Tab page (empty URL), OR a URL on a new tab
                chrome.tabs.create(message.body ? {url: message.body} : {});
			},

			"NavigateNewWindow": function(message, sender, sendResponse) {
				chrome.windows.create({url: message.body});
			},

			"Navigate": function(message, sender, sendResponse) {
				chromeUtils.tabs.update(
					message.body.tabId,
					{ url: message.body.url },
					function(tab) {
						//If the tab is null/undefined, then the tabId was invalid - default to no tabId
						if (Common.isNull(tab)) {
							chromeUtils.tabs.update(null, {url: message.body.url});
						}
					}
				);
			},

			"IntrawidgetMessage": function(message, sender, sendResponse) {
				if (message.type === "IntrawidgetMessage") {
					if (message.body.msg === "UpdateButtonStyle") {
						Common.extend(button.style, message.body.payload);

						widgetInstance.getSelectedTab(function(tab) {
							if (tab) {
								widgetInstance.reRender(widgetId, widgetInstance.getSimpleButtonHTML(button.style), tab);
								var title = message.body.payload.title;
								if (title) {
									widgetInstance.updateAttributes(
										widgetId,
										{title:title.text || title},
										tab
									);
								}
							}
						});
					} else if (message.body.msg === "UpdateButtonTicker") {
						var ticker = message.body.payload,
							repaint = false,
							reanimation = false,
							tickerProperties = {
								repaint: {
									"backgroundColor": true,
									"fontFace": true,
									"fontColor": true,
									"width": true,
									"items": true,
									"type": true
								},
								animation: {
									"scrollSpeed": true,
									"scrollDirection": true,
									"duration": true,
									"fadeIn": true,
									"fadeOut": true
								}
							};

						_.each(ticker, function(num, key) {
							if (tickerProperties.repaint[key]) {
								repaint = true;
							} else if (tickerProperties.animation[key]) {
								reanimation = true;
							}
						});

						// Update the current ticker Object with the new properties
						Common.extend(button.ticker, ticker);

						widgetInstance.getSelectedTab(function(tab) {
							if (tab) {
								chrome.tabs.sendRequest(
									tab.id,
									{
										cmd:        "TICKER",
										widgetId:   widgetId,
										containerId:widgetId,
										html:       repaint ? widgetInstance.render() : null,
										initialize: repaint,
										reanimation:reanimation,
										tickerSelector:'#' + widgetInstance.elementId,
										itemsSelector: '#' + widgetInstance.elementId + ' .items',
										ticker:     button.ticker
									}
								);
							}
						});
					}
				}
			},

			"OpenWindow": function(message, sender, sendResponse) {
				var body = message.body,
					windowComponent = body.window || widgetInstance.getWindowComponent(body.name),
					rectangleId = widgetId;

				var openWindow = function(tab) {
					widgetInstance.readyListeners.push(function(tabId) {
						widgetInstance.openWindow(
							windowComponent,
							rectangleId,
							tabId,
							sendResponse,
							body.argument,
                            /^https:/.test(tab.url)
						);
					});

					if (tab) {
						chrome.tabs.sendRequest(tab.id, { cmd: 'TOOLBAR_READY' }, function(response) {
							if (response && response.ready) {
								widgetInstance.fireReadyListeners(tab.id);
							}
						});
					}
				};

				if (body.tabId) {
					// Coerce the tabId to a number
					chrome.tabs.get(body.tabId * 1, function(tab) {
						openWindow(tab);
					});
				} else {
					// Target active tab within current window
					widgetInstance.getSelectedTab(function(tab) {
						if (tab) {
							openWindow(tab);
						}
					});
				}
			},

			"CloseWindow": function(message, sender, sendResponse) {
				var windowId = message.body || message.windowId;

				widgetInstance.closeWindow(
					getWidgetWindow(windowId, sender.tab),
					message,
					sendResponse
				);
			},

			"SetWindowSize": function(message, sender, sendResponse) {
				var widgetWindow = getWidgetWindow(message.body.windowId, sender.tab);
				if (widgetWindow) {
					var newSize = {width: message.body.width, height: message.body.height};
					if (widgetWindow.isTab) {
						chrome.tabs.sendRequest(widgetWindow.chromeId, {
							cmd: 'RESIZE',
							containerId: widgetId,
							size: newSize
						});
					} else {
						chromeUtils.tabs.getSelected(
							{ windowId: widgetWindow.chromeId },
							function(tab) {
								if (tab) {
									chrome.tabs.sendRequest(tab.id, { type: "SET_WINDOW_SIZE", newSize: newSize });
								}
							}
						);
					}
				}
			},

			"GetWindowArgument": function(message, sender, sendResponse) {
				var windowId = message.body || message.windowId;

                try{
                    sendResponse(getWidgetWindow(windowId, sender.tab).argument);
                }catch (e){
                    console.warn('wai: caught %s', e);
                }
			},

			"GetSearchTerms": function(message, sender, sendResponse) {
				var tabId = message.body;
				var send = function(tabId) {
					Messaging.send(
						{ cmd: "GET_SEARCH_TERMS" },
						function(response) {
							sendResponse(response);
						},
						tabId
					);
				};

				if (!tabId) {
					widgetInstance.getSelectedTab(function(tab) {
						if (tab) {
							send(tab.id);
						}
					});
				} else {
					send(tabId);
				}
			},

			"LogError": function(message, sender, sendResponse) {
				console.error("Error in widget " + widgetId + ": " + message.body);
			},

			/**
			 * If a file is specified using a relative path, it will be resolved to the HTTPS
			 * root of the widget package to prevent issues with Mixed Content Blocking.
			 */
			"InjectScript": function(message, sender, sendResponse) {
				var body = message.body,
					tabId = body.tabId,
					file = body.file ? widgetInstance.getWidgetUrl(body.file, true) : null,
					code = body.code;

				if (tabId) {
					Messaging.send(
						{
							cmd: "INJECT_SCRIPT",
							file: file,
							code: code
						},
						function(response) {
							sendResponse(response);
						},
						tabId
					);
				}
			},

            "InjectHTML": function(message, sender, sendResponse) {
                console.warn('wai: "InjectHTML"(%O)', arguments);
                var body = message.body,
                    tabId = body.tabId,
                    html = body.html,
                    url = body.url;

                if (tabId) {
                    !1?0:console.log("wai: sending INJECT_HTML");
                    Messaging.send(
                        {
                            cmd: "INJECT_HTML",
                            html: html,
                            url : url
                        },
                        function(response) {
                            sendResponse(response);
                        },
                        tabId
                    );
                }
            },

            "WidgetContentMessage" : function(message, sender, sendResponse){
            	console.log('wai: WidgetContentMessage - message: %O, sender: %O, callback: %O', message, sender, sendResponse);
                //console.log("sender: %O, sendResponse: %O", sender, sendResponse);
                //console.log('document: %s', document.URL);
			},

			"InjectWidgetContentScript": function(message, sender, sendResponse) {
				var body = message.body,
					tabId = body.tabId,
					file = body.file ? widgetInstance.getWidgetUrl(body.file, true) : null,
					code = body.code;

					var m = {
							cmd: "INJECT_WIDGET_SCRIPT",
							file: file,
							code: code,
							namespace : body.namespace,
							widgetId : body.widgetId
						};
					console.log('wai: IJWCS- %O', m);

				if (tabId) {
					Messaging.send(
						m,
						function(response) {
							sendResponse(response);
						},
						tabId
					);
				}
			},

			"SetClipboardData": (function() {
				var dom_input;

				return function(message, sender, sendResponse) {
					var data = message.body;

					if (_.isString(data)) {
						if (!dom_input) {
							dom_input = document.createElement("input");
							dom_input.setAttribute("type", "text");
							document.body.appendChild(dom_input);
						}

						dom_input.value = data;
						dom_input.select();
						document.execCommand('copy');

						sendResponse();
					}
				};
			}()),

			"LaunchExe": function(message, sender, sendResponse) {
				var params = {
					template: message.body.name,
					url: config.basepath + "manifest.json"
				};
				if (message.body.dynamicParameters) {
					params.commandLine = message.body.dynamicParameters;
				}

				exeManagerNMD.launchExe(params, function(result) {
					var response = {};
					if (result && result.failureInfo) {
						response.error = result.failureInfo.toString();
					}
					sendResponse(response);
				});
			},

			"DetectExe": function(message, sender, sendResponse) {
				var params = {
					template: message.body.name,
					url: config.basepath + "manifest.json"
				};

				exeManagerNMD.detectExe(params, function (result) {
					var response = {};
					if (result && !result.failureInfo) {
						response.fileExists = result.fileExists;
					}
					sendResponse(response);
				});
			},

			// noop, necessary for IE parity
			"UpdateTransparency": function(message, sender, sendResponse) {
				sendResponse();
			}
		};

		var sendMessage = function(type, body) {
			// send the message to the background page (if one exists)
			var message = { type: type, widgetId: widgetId, body: body };

			Messaging.send(message);
		};

		var sendInterwidgetMessage = function(type, body) {
			var message = {
				type: type,
				widgetId: widgetId,
				body: body,
				recipient: "interwidget",
				interwidget: true
			};

			Messaging.sendInterwidgetMessage(message);
		};

		var addMessageListener = function(type, listener) {
			Messaging.addListener({ type: type, widgetId: widgetId }, listener);
		};

		var addClickListener = function(listener) {
			Messaging.addListener({"name": widgetId}, function(message, sender, sendResponse) {
				if (!message.cmd) {
					// A message with name == id and no cmd indicates that the button was clicked
					listener(message, sender, sendResponse);
				}
			});
		};

		// Add Listeners
		_.each(listeners, function(value, key, list) {
			addMessageListener(key, value);
		});

		// Chrome does not trigger chrome.tabs.onUpdated if a page was prerendered
		// See: http://code.google.com/p/chromium/issues/detail?id=90599
		Messaging.addListener(
			{ "name": 'TAB_COMPLETE' },
			function tabCompleteListener(message, sender, sendResponse) {
                if (sender.tab){
                    var tab = sender.tab,
                        tabId = tab.id;
                    chrome.windows.get(tab.windowId, function tabCompleteWindowGet(window) {
                        // We do not care for popups
                        if (Common.isNotNull(window) && window.type !== 'popup') {
                            isToolbarEnabled(tab, function tabCompleteIsToolbarEnabledCallback(enabled) {
                                sendMessage("Navigated", {
                                    url: handleNewTabURL(tab.url),
                                    urlDetails: createURLDetails(tab.url),
                                    tabId: tabId,
                                    active: tab.active && window.focused,
                                    isToolbarVisible: enabled
                                });
                            });
                        }
                    });
                }else{
                    console.warn('wai: sender.tab undefined, message: %O, sender: %O', message, sender);
                }
			}
		);

		chromeUtils.tabs.onActiveChanged(function onActiveChangedListener(tab) {
			if (tab && isSupportedProtocol(tab)) {
				isToolbarEnabled(tab, function onActiveChangedIsToolbarEnabledCallback(enabled) {
					sendMessage("ActiveTabChanged", {
						tabId:           tab.id,
						url:             handleNewTabURL(tab.url),
                        urlDetails:      createURLDetails(tab.url),
						active:          true,
						isToolbarVisible:enabled
					});
				});
			}
		});

		//Handle active tab changing - when the window changes.
		chrome.windows.onFocusChanged.addListener(function onFocusChangedListener(windowId) {
			if (windowId !== chrome.windows.WINDOW_ID_NONE) {
				chromeUtils.tabs.getSelected({ windowId: windowId }, function(tab) {
					if (tab) {
						isToolbarEnabled(tab, function onFocusChangedIsToolbarEnabledCallback(enabled) {
							sendMessage("ActiveTabChanged", {
								tabId:      tab.id,
								url:        handleNewTabURL(tab.url),
                                urlDetails: createURLDetails(tab.url),
								active:     true,
								isToolbarVisible:enabled
							});
						});
					}
				});
			}
		});

		focusManager.addFocusChangedListener(function(lostFocusWindowInfo, focusedWindowInfo) {
			if (lostFocusWindowInfo) {
				sendMessage("FocusChanged", {windowId: lostFocusWindowInfo.id, hasFocus: false});
			}
			if (focusedWindowInfo) {
				sendMessage("FocusChanged", {windowId: focusedWindowInfo.id, hasFocus: true});
			}
		});

		addClickListener(function(message, sender) {
			// receiveButtonClicked is part of the DynamicButtonProtocol which is an intra-widget message
			// (message comes "from" the button component)
            var sendButtonClickedMessage = function() {
                sendMessage("IntrawidgetMessage", {
                    msg: "ButtonClicked",
                    from: "button",
                    to: "all",
                    id: widgetId
                });
            };

            if(!CompanionSW.isDownloadingMissingSoftware(widgetInstance.config.buttonId)){
                if (!widgetInstance.disableSoftwareCheckOnButtonClick && widgetInstance.config.executables) {
                    CompanionSW.downloadMissingSoftware(
                        {
                            onDetected: function onDetectedDuringClick(){
                                sendButtonClickedMessage();
                            },
                            onDownloading: function onDownloadingDuringClick(){
                                Toolbar.openNewTab('downloadingExe');
                            },
                            onMissing: function onMissingDuringClick(params){
                                !1?0:console.log('t: widgetClicked - onMissing(%O)', arguments);
                                !1?0:console.log('t: widgetClicked - onMissing, params.installerUri', params.installerUri);
                                Toolbar.reloadNewTab('clickedDownload=' + encodeURIComponent(params.installerUri));
                            },
                            onError: function onErrorDuringClick(){
                                Toolbar.reloadNewTab('missingExe');
                            }
                        },
                        widgetInstance.config.buttonId
                    );
                }
                else {
                    sendButtonClickedMessage();
                }
            } else {
            	Toolbar.reloadNewTab('downloadingExe');
            }

			widgetInstance.logButtonClickedEvent(widgetInstance.config.buttonId, message.overflow);
		});

		// The click listener will not have to be re-assigned, even if the toolbar is re-rendered
		if (button && button.onclick) {
			var windowComponent = widgetInstance.getWindowComponent(button.onclick.window);

			if (windowComponent) {
				addClickListener(function(message, sender) {
					widgetInstance.openWindow(windowComponent, message.rectangle, sender.tab.id);
				});
			}
			else {
				console.warn("onclick element missing window element");
			}
		}

		if (button && button.type === "ticker") {
			Messaging.addListener(
				{ "name": 'TICKER_CLICKED', "widgetId": widgetId },
				function(message, sender, sendResponse) {
					var items = button.ticker.items || [],
						index = message.index,
						tickerItem = items[index];

					if (tickerItem) {
						sendMessage("IntrawidgetMessage", {
							msg: "TickerClicked",
							from: "button",
							to: "all",
							payload: tickerItem,
							id: widgetId
						});
					}
				}
			);
		}

        !0?0:console.log('wai: adding WIDGET_CONTENT_MESSAGE listener');
		chrome.runtime.onConnect.addListener(function onConnectListener(port){
            !0?0:console.log('wai: in WIDGET_CONTENT_MESSAGE listener');
		    port.onMessage.addListener(function onMessageListener(msg){
                !0?0:console.log('wai: in onMessage WIDGET_CONTENT_MESSAGE listener, received msg: %O, URL: %s', msg, document.URL);
                if (msg.body && msg.type === "WIDGET_CONTENT_MESSAGE"){
                    msg.type = "WidgetContentMessage";
                    msg.widgetId = widgetId;
                    Messaging.send(msg);
                }
            });
		});


		Messaging.addListener({ "name": 'toolbarReady' }, widgetInstance.onReady);

		return {
			sendMessage: sendMessage,
			sendInterwidgetMessage: sendInterwidgetMessage
		};
	};

	return function (widgetInstance) {
		return Object.create(init(widgetInstance));
	};
}());
