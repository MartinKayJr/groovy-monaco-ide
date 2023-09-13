<template>
  <div id="editor" style="width: 500px; height: 500px"></div>
</template>

<script>
import { listen } from "vscode-ws-jsonrpc";

window.setImmediate = window.setTimeout; 

import {
  MonacoLanguageClient,
  CloseAction,
  ErrorAction,
  createConnection,
  MonacoServices,
} from "monaco-languageclient";

const ReconnectingWebSocket = require('reconnecting-websocket');

import loader from "@monaco-editor/loader";

export default {
  name: "Editor",
  async mounted() {
    loader.init().then((monaco) => {


      const fileurl =  monaco.Uri.parse('file:///F:/tmp/lsp/demo.groovy')
      monaco.languages.register({ 
        id: 'groovy',
        extensions: [".groovy"],
        aliases: ["groovy"]
      })
          monaco.languages.setMonarchTokensProvider('groovy', {
          tokenizer: {
              root: [
                  // 注释
                  [/(\/\/.*)|(\/\*[\s\S]*?\*\/)/, 'comment'],

                  // 字符串
                  [/"([^"\\]|\\.)*$/, 'string.invalid'], // 未闭合的字符串
                  [/"/, 'string', '@string'],

                  // 关键字
                  [
                      /(def|if|else|for|while|switch|case|break|continue|return|try|catch|finally|throw|assert|import|package|class|interface|extends|implements|new|this|super|static|final|abstract|native|synchronized|volatile|transient|public|protected|private|void|boolean|byte|char|short|int|long|float|double)\b/,
                      'keyword'
                  ],

                  // 标识符
                  [/[a-zA-Z_$][a-zA-Z0-9_$]*/, 'identifier'],

                  // 数字
                  [/\d*\.\d+([eE][-+]?\d+)?/, 'number.float'],
                  [/\d+/, 'number'],

                  // 操作符和分隔符
                  [/[{}()[]]/, '@brackets'],
                  [/[+\-*/%=&|~^<>!]/, 'operator'],

                  // 空白字符
                  { include: '@whitespace' }
              ],

              string: [
                  [/[^\\"]+/, 'string'],
                  [/"/, 'string', '@pop']
              ],

              whitespace: [
                  [/[ \t\r\n]+/, 'white'],
                  [/\s+/, 'white']
              ]
          }
      })
      const model =  monaco.editor.createModel('' , 'groovy' ,  fileurl)
        
      const editorOptions = {
        model,
        theme: 'vs-dark',
        minimap: { enabled: false },
        automaticLayout: true,
        suggestOnTriggerCharacters: true,
      };

      monaco.editor.create(document.getElementById("editor"), editorOptions);

      try {
        MonacoServices.get();
      } catch (err) {
        MonacoServices.install(monaco, { rootUri: 'file:///F:/tmp'});
      }
      this.connectToLangServer();

    });

  },
  methods: {
    createLanguageClient: function (connection) {

      return new MonacoLanguageClient({
        name: "Groovy language client",
        clientOptions: {
          documentSelector: ["groovy"],
          errorHandler: {
            error: () => ErrorAction.Continue,
            closed: () => CloseAction.Restart,
          },
        },

        connectionProvider: {
          get: (errorHandler, closeHandler) => {
            return Promise.resolve(
              createConnection(connection, errorHandler, closeHandler)
            );
          },
        },
      });
    },
    createWebSocket: function(url) {
      const socketOptions = {
          maxReconnectionDelay: 10000,
          minReconnectionDelay: 1000,
          reconnectionDelayGrowFactor: 1.3,
          connectionTimeout: 10000,
          maxRetries: Infinity,
          debug: true
      };
      return new ReconnectingWebSocket(url, [], socketOptions);
    },
    connectToLangServer: function () {
      const webSocket = this.createWebSocket("ws://192.168.1.24/groovy");

      listen({
        webSocket: webSocket,
        onConnection: (connection) => {
          var languageClient = this.createLanguageClient(connection);
          var disposable = languageClient.start();

          connection.onClose(function () {
            return disposable.dispose();
          });
          
          connection.onError(function (error) {
            console.log(error);
          });

        },
      });
    },
  },
};
</script>


