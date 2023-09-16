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

        keywords: [
          'abstract',
          'continue',
          'for',
          'new',
          'switch',
          'assert',
          'goto',
          'do',
          'if',
          'private',
          'this',
          'break',
          'protected',
          'throw',
          'else',
          'public',
          'enum',
          'return',
          'catch',
          'try',
          'interface',
          'static',
          'class',
          'finally',
          'const',
          'super',
          'while',
          'true',
          'false',
        ],

        typeKeywords: [
          'boolean',
          'double',
          'byte',
          'int',
          'short',
          'char',
          'void',
          'long',
          'float',
        ],

        operators: [
          '=',
          '>',
          '<',
          '!',
          '~',
          '?',
          ':',
          '==',
          '<=',
          '>=',
          '!=',
          '&&',
          '||',
          '++',
          '--',
          '+',
          '-',
          '*',
          '/',
          '&',
          '|',
          '^',
          '%',
          '<<',
          '>>',
          '>>>',
          '+=',
          '-=',
          '*=',
          '/=',
          '&=',
          '|=',
          '^=',
          '%=',
          '<<=',
          '>>=',
          '>>>=',
        ],

        // we include these common regular expressions
        symbols: /[=><!~?:&|+\-*/^%]+/,

        // C# style strings
        escapes: /\\(?:[abfnrtv\\"']|x[0-9A-Fa-f]{1,4}|u[0-9A-Fa-f]{4}|U[0-9A-Fa-f]{8})/,

        // The main tokenizer for our languages
        tokenizer: {
          root: [
            // identifiers and keywords
            [
              /[a-z_$][\w$]*/,
              {
                cases: {
                  '@typeKeywords': 'keyword',
                  '@keywords': 'keyword',
                  '@default': 'identifier',
                },
              },
            ],
            [/[A-Z][\w$]*/, 'type.identifier'], // to show class names nicely

            // whitespace
            { include: '@whitespace' },

            // delimiters and operators
            [/[{}()[\]]/, '@brackets'],
            [/[<>](?!@symbols)/, '@brackets'],
            [
              /@symbols/,
              {
                cases: {
                  '@operators': 'operator',
                  '@default': '',
                },
              },
            ],

            // @ annotations.
            // As an example, we emit a debugging log message on these tokens.
            // Note: message are supressed during the first load -- change some lines to see them.
            [
              /@\s*[a-zA-Z_$][\w$]*/,
              { token: 'annotation', log: 'annotation token: $0' },
            ],

            // numbers
            [/\d*\.\d+([eE][-+]?\d+)?/, 'number.float'],
            [/0[xX][0-9a-fA-F]+/, 'number.hex'],
            [/\d+/, 'number'],

            // delimiter: after number because of .\d floats
            [/[;,.]/, 'delimiter'],

            // strings
            [/("|')([^("|')\\]|\\.)*$/, 'string.invalid'], // non-teminated string
            [/("|')/, { token: 'string.quote', bracket: '@open', next: '@string' }],

            // characters
            [/'[^\\']'/, 'string'],
            [/(')(@escapes)(')/, ['string', 'string.escape', 'string']],
            [/'/, 'string.invalid'],
          ],

          comment: [
            [/[^/*]+/, 'comment'],
            [/\/\*/, 'comment', '@push'], // nested comment
            ['\\*/', 'comment', '@pop'],
            [/[/*]/, 'comment'],
          ],

          string: [
            [/[^\\("|')]+/, 'string'],
            [/@escapes/, 'string.escape'],
            [/\\./, 'string.escape.invalid'],
            [/("|')/, { token: 'string.quote', bracket: '@close', next: '@pop' }],
          ],

          whitespace: [
            [/[ \t\r\n]+/, 'white'],
            [/\/\*/, 'comment', '@comment'],
            [/\/\/.*$/, 'comment'],
          ],

        },

          
      })
      monaco.languages.onLanguage('groovy', function() {
        monaco.languages.setLanguageConfiguration('groovy', {
            comments: {
                lineComment: '//',
                blockComment: ['/*', '*/'],
            },
            brackets: [
                ['{', '}'],
                ['(', ')']
            ],
        })
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


