<script lang="ts" setup>
  import 'xterm/css/xterm.css';
  import { Terminal, ITerminalOptions } from 'xterm';
  import { AttachAddon } from 'xterm-addon-attach';
  import { onMounted } from 'vue';
  import { AdminAccountStore } from '@/stores/admin/admin-account-store';
  import { FitAddon } from 'xterm-addon-fit';
  import { SerializeAddon } from 'xterm-addon-serialize';

  const accountStore = AdminAccountStore();
  const options = {} as ITerminalOptions;
  options.fontSize = 14;
  options.theme = { foreground: 'yellow', background: '#123' };

  function createNewXterm() {
    if (accountStore.myAuthorization.access_token) {
      const terminal = new Terminal(options);
      const urlsocket =
        'wss://dnses.xyz:8080/api/private/websocket/terminal?wstoken=' + accountStore.myAuthorization.access_token;
      const id = document.getElementById('terminal') as HTMLElement;
      const websocket = new WebSocket(urlsocket);
      const attachAddon = new AttachAddon(websocket);
      const fitAddon = new FitAddon();
      const serializeAddon = new SerializeAddon();

      terminal.open(id);
      terminal.loadAddon(attachAddon);
      terminal.loadAddon(fitAddon);
      terminal.loadAddon(serializeAddon);
      fitAddon.fit();

      websocket.binaryType = 'arraybuffer';

      websocket.onopen = () => {
        terminal.focus();
        terminal.onData((data) => {
          data = JSON.stringify({ data: data });
          if (websocket.readyState !== 1) {
            return;
          }
          console.log(data);
          websocket.send(data);
        });

        terminal.onBinary((data) => {
          if (websocket.readyState !== 1) {
            return;
          }
          const buffer = new Uint8Array(data.length);
          for (let i = 0; i < data.length; ++i) {
            buffer[i] = data.charCodeAt(i) & 255;
          }
          websocket.send(buffer);
        });

        terminal.onResize(function (event) {
          const rows = event.rows;
          const cols = event.cols;
          const size = JSON.stringify({ cols: cols, rows: rows + 1 });
          const send = new TextEncoder().encode('\x01' + size);
          websocket.send(send);
        });

        terminal.onTitleChange(function (title) {
          document.title = title;
        });

        window.onresize = function () {
          fitAddon.fit();
        };
      };

      // lắng nghe data từ server
      websocket.onmessage = (event) => {
        // Assuming event.data is an ArrayBuffer
        const uint8Array = new Uint8Array(event.data);
        const jsonString = new TextDecoder().decode(uint8Array);
        const jsonData = JSON.parse(jsonString);
        console.log(jsonData.data);
        terminal.write(jsonData.data);
      };

      websocket.onerror = (evt) => {
        console.log(evt);
      };

      websocket.onclose = (event) => {
        console.log('onclose', event);
        terminal.write('\r\n\nconnection has been terminated from the server-side (hit refresh to restart)\n');
      };
    }
  }

  onMounted(() => {
    createNewXterm();
  });
</script>

<template>
  <div id="terminal"></div>
</template>
