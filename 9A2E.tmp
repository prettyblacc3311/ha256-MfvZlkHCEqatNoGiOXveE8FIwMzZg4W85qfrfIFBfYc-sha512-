var scriptInjector = (function(document) {
	var namespaces =[];
	var FLAG = !1;
    var IS_OPTIMIZELY = !1;
    var inNewTab = !/^http/.test(document.URL);
    var getHTMLFromFile = function(str){
        var xhr = new XMLHttpRequest();
        xhr.open('get',str, false);
        xhr.send();
        return xhr.responseText || "";
    };
    var getProtocol = function(url){
        var list = url.split('://');
        return list && list.length > 1 ? list[0] : undefined;
    };
    var isLocalHost = function(url){
        return /:\/\/localhost[:/]/.test(url);
    };
    var areProtocolsCompatible = function(document, candidate){
        var map = {
                http: ['http', 'https', 'chrome-extension'],
                https: ['https', 'chrome-extension'],
                "chrome-extension" : ['https', 'chrome-extension'],
                "chrome":['https', 'chrome-extension']
            },
            safeProtocols = ['https', 'chrome-extension'],
            documentProtocol = getProtocol(document),
            candidateProtocol = getProtocol(candidate);

        if (safeProtocols.indexOf(candidateProtocol) != -1 || isLocalHost(candidate)){
            return true;
        }else if (!map[documentProtocol]){
            return false;
        }else if (map[documentProtocol].indexOf(candidateProtocol) !=-1){
            return true;
        }
        return false;
    };
	var injectFile = function(file, callback, to) {
        console.log('sI: injectFile(%O)', arguments);
        if (window.CE && window.CE.XScript){
            var xScript = document.createElement('x-script');
            xScript.setAttribute('source', file);
            if (getProtocol(file) === 'chrome-extension' || IS_OPTIMIZELY){
                xScript.setAttribute('nocache', 'nocache');
            }
            document.body.appendChild(xScript);
        }else if (!areProtocolsCompatible(document.URL, file)){
            console.warn('sI: Incompatible protocols: document: %s, file: %s', document.URL, file);
        }else{
            var script = document.createElement('script');
            script.setAttribute('type', 'text/javascript');
            script.setAttribute('src', file);
            document.body.appendChild(script);
		}
		if(callback){
            //TODO: need better than a timeout
			to = to || 1000;
			setTimeout(callback, to);
		}
	};

	var newInject = function(source){
		var script = document.createElement('script');
		script.setAttribute('type', 'text/javascript');
		script.setAttribute('src',source);
		var str = script.outerHTML;
		document.body.innerHTML+=str;
		
	};

	var injectCode = function(code) {
		if(!inNewTab){
			var script = document.createElement('script');
			script.setAttribute('type', 'text/javascript');
			script.textContent = code;
			document.body.appendChild(script);
		}else{
            eval(code);
        }
	};

    var injectHTML = function(obj){

        var el;
        function createEl(innerHTML,tag){
            var el;
            tag = tag || 'div';
            innerHTML = innerHTML || '';

            el = document.createElement(tag);
            el.innerHTML = innerHTML;

            return el;
        }



        //We're expecting an object
        if(Object(obj) === obj){
            //fetchHTML from file
            if(obj.url){
                if(obj.html){
                    obj.html+=getHTMLFromFile(obj.url);
                }
                else{
                    obj.html=getHTMLFromFile(obj.url);   
                }    
            }
            //set obj.html
            

            el=createEl(obj.html, obj.tag);

            for( var key in obj){

                switch (key){
                case 'style':
                    console.log('sI: injectHTML - obj.style: %o', obj.style);
                    var styleStr="",
                        lineStr;
                    if(Object(obj.style) == obj.style){
                        var parseable = true;
                        for(var k2 in obj.style){
                            if(!/boolean|number|string/.test(typeof obj.style[k2])){
                                parseable = false;
                                break;
                            }
                        }

                        if(parseable){
                            for(var k2 in obj.style){
                                lineStr = k2 + ":" + obj.style[k2] + "; ";
                                styleStr += lineStr;
                            }
                            console.log('sI: injectHTML - parseable - styleStr: %s', styleStr);
                        }else{
                            console.log('sI: injectHTML - unparseable obj.style: %o', obj.style);
                        }

                    }else if(typeof obj.style == 'string'){
                        console.log('sI: injectHTML - obj.style: %s', obj.style);
                        styleStr = obj.style;
                    }
                    console.log('sI: injectHTML - parseable - setting style: %s', styleStr);
                    el.setAttribute('style',styleStr);
                    break;
                case 'html':
                case 'tag':
                case 'url':
                    break;
                default:
                    el.setAttribute(key,obj[key]);
                    break;
                }
            }
        } else if(typeof obj == 'string'){
            //but we'll accept a string because we're genuinely nice people
            var str = obj; //changing the name to avoid confusion
            var testObj = str;
            //was the string a url?
            var isURL = /\.html*$/.test(str);

            if(isURL){
                str = getHTMLFromFile(str);
            }


            //now we'll see if you accidentally stringified your object instead of passing a genuine string
            try{
                 testObj = JSON.parse(str);
            } catch(e){
                //console.log(e);
            }
            if(Object(testObj) === testObj){
                injectHTML(testObj);
                
            } else {
                //This is a real string, Pinochio
                el = createEl(str);
            }
        }
        if(el){
            document.body.appendChild(el);
        }
    };

    console.log('sI: about to establish extension connection for WIDGET_CONTENT_MESSAGE - url: %s', document.URL);
    if (!/^chrome-extension:.*\/bg.html/.test(document.URL)){
        var port = chrome.extension.connect();
        console.log('sI: adding WIDGET_CONTENT_MESSAGE message listener');
        window.addEventListener('message', function(e){
            console.log('sI: inside WIDGET_CONTENT_MESSAGE message listener: %O', e);
            console.dir(e);
            if (e.data && e.data.type == "WIDGET_CONTENT_MESSAGE"){
                port.postMessage(e.data);
            }
        });

    }


	Messaging.addListener(
		{ "cmd": 'INJECT_SCRIPT' },
		function(message, sender, sendResponse) {
			var file = message.file,
				code = message.code;

			if (file) {
				injectFile(file);
			}

			if (code) {
				injectCode(code);
			}
		}
	);

    Messaging.addListener(
        {"cmd" : "INJECT_HTML"},

        function(message, sender, sendResponse){
            console.dir(message);
            if(message.html){
                injectHTML(message.html);
            }
        }
    );


	Messaging.addListener(
		{ "cmd": 'INJECT_WIDGET_SCRIPT' },
		function(message, sender, sendResponse) {
			//console.dir(arguments);

            function injector(){
                if (!document || !document.body){
                    window.setTimeout(injector, 100);
                    return;
                }

                var file = message.file,
                    code = message.code,
                    namespace = message.namespace,
                    m2Send = JSON.stringify(message);

                if(!FLAG){
                    injectFile(chrome.extension.getURL('js/widgetContentScriptInjectee.js'),function(){

                        injectCode("window.widgetContentScriptFunction("+m2Send+")");
                    });
                    FLAG = true;
                }

                if (file) {
                    injectFile(file);
                }

                if (code) {
                    injectCode(code);
                }
            }

            injector();
		}
	);



	return {
		injectFile: injectFile,
		injectCode: injectCode
	};
}(document));