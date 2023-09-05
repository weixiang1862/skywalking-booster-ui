<!-- Licensed to the Apache Software Foundation (ASF) under one or more
contributor license agreements.  See the NOTICE file distributed with
this work for additional information regarding copyright ownership.
The ASF licenses this file to You under the Apache License, Version 2.0
(the "License"); you may not use this file except in compliance with
the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License. -->
<template>
  <div class="dashboard-tool">
    <div class="flex-h">
      <div class="selectors-item">
        <span class="label">$Service: </span>
        <Selector
          size="small"
          :value="state.service.value"
          :options="arthasStore.services"
          placeholder="Select a service"
          @change="changeField('service', $event)"
        />
      </div>
      <div class="selectors-item">
        <span class="label">$ServiceInstance: </span>
        <Selector
          class="instance"
          size="small"
          :value="state.instance.value"
          :options="arthasStore.instances"
          placeholder="Select a instance"
          @change="changeField('instance', $event)"
        />
      </div>
      <div class="selectors-item">
        <span class="label">$Arthas Tunnel Server: </span>
        <input v-model="arthasTunnelServer" class="tunnel-server" />
      </div>
      <div class="btn-item">
        <el-button v-loading.fullscreen.lock="loading" size="small" type="primary" @click="startConnect(false)">
          Connect
        </el-button>
      </div>
      <div class="btn-item">
        <el-button size="small" type="primary" @click="disconnect">Disconnect</el-button>
      </div>
    </div>
  </div>
  <div id="terminal" class="w-full h-full" />
</template>
<script setup lang="ts">
  import { reactive, ref } from "vue";
  import { useArthasStore } from "@/store/modules/arthas";
  import { Terminal } from "xterm";
  import { ElMessage } from "element-plus";
  import "xterm/css/xterm.css";
  import axios from "axios";
  import { cancelToken } from "@/utils/cancelToken";
  /*global Recordable */
  const arthasStore = useArthasStore();
  const state = reactive<Recordable>({
    service: { value: "", label: "" },
    instance: { value: "", label: "" },
  });
  const loading = ref(false);
  let arthasTunnelServer = "ws://127.0.0.1:7777/ws";

  let ws: WebSocket | undefined;
  let intervalReadKey = -1;
  const DEFAULT_SCROLL_BACK = 1000;
  const MAX_SCROLL_BACK = 9999999;
  const MIN_SCROLL_BACK = 1;

  let xterm = new Terminal({ allowProposedApi: true });

  init();

  async function init() {
    await getServices();
  }

  async function getServices() {
    const resp = await arthasStore.getServices("GENERAL");
    if (resp.errors) {
      ElMessage.error(resp.errors);
      return;
    }
    state.service = arthasStore.services[0];
    getInstances(state.service.id);
  }

  async function getInstances(id?: string) {
    const resp = await arthasStore.getInstances(id);
    if (resp.errors) {
      ElMessage.error(resp.errors);
      return;
    }
    state.instance = arthasStore.instances[0];
  }

  function changeField(type: string, opt: any) {
    state[type] = opt[0];
    if (type === "service") {
      getInstances(state.service.id);
    }
  }

  /** init websocket **/
  function initWs(silent: boolean) {
    let uri = `${arthasTunnelServer}?method=connectArthas&id=${state.instance.value}`;
    ws = new WebSocket(uri);
    ws.onerror = function () {
      ws ?? ws!.close();
      ws = undefined;
      !silent && ElMessage.error("Connect error");
    };
    ws.onopen = function () {
      const { cols, rows } = initXterm("0");
      xterm.onData(function (data) {
        ws?.send(JSON.stringify({ action: "read", data: data }));
      });
      ws!.onmessage = function (event: MessageEvent) {
        if (event.type === "message") {
          var data = event.data;
          xterm.write(data);
        }
      };
      ws?.send(JSON.stringify({ action: "resize", cols, rows }));
      intervalReadKey = window.setInterval(function () {
        if (ws != null && ws.readyState === 1) {
          ws.send(JSON.stringify({ action: "read", data: "" }));
        }
      }, 30000);
    };
    ws.onclose = function (message) {
      if (intervalReadKey != -1) {
        window.clearInterval(intervalReadKey);
        intervalReadKey = -1;
      }
      if (message.code === 2000) {
        ElMessage.error(message.reason);
      }
    };
  }

  /** init xterm **/
  function initXterm(scrollback: string) {
    let scrollNumber = parseInt(scrollback, 10);
    xterm = new Terminal({
      rows: 40,
      cols: 150,
      screenReaderMode: false,
      convertEol: true,
      allowProposedApi: true,
      scrollback: isValidNumber(scrollNumber) ? scrollNumber : DEFAULT_SCROLL_BACK,
    });

    xterm.open(document.getElementById("terminal")!);
    return {
      cols: xterm.cols,
      rows: xterm.rows,
    };
  }

  function isValidNumber(scrollNumber: number) {
    return scrollNumber >= MIN_SCROLL_BACK && scrollNumber <= MAX_SCROLL_BACK;
  }

  const connectGuard = (silent: boolean): boolean => {
    if (state.instance.value == "") {
      if (silent) {
        return false;
      }
      ElMessage.error("AgentId can not be empty");
      return false;
    }
    if (ws) {
      ElMessage.error("Already connected");
      return false;
    }
    return true;
  };

  function startConnect(silent: boolean = false) {
    if (connectGuard(silent)) {
      loading.value = true;
      axios
        .post(
          "/arthas/start",
          {
            serviceName: state.service.value,
            instanceName: state.instance.value,
          },
          { cancelToken: cancelToken() },
        )
        .then(() => {
          setTimeout(() => {
            loading.value = false;
            initWs(silent);
          }, 5000);
        })
        .catch((err: Error) => {
          loading.value = false;
          throw err;
        });
    }
  }

  function disconnect() {
    axios
      .post(
        "/arthas/stop",
        {
          serviceName: state.service.value,
          instanceName: state.instance.value,
        },
        { cancelToken: cancelToken() },
      )
      .then(() => {
        try {
          ws!.close();
          ws!.onmessage = null;
          ws!.onclose = null;
          ws = undefined;
          xterm.dispose();
        } catch {
          ElMessage.error("No connection, please start connect first.");
        }
      })
      .catch((err: Error) => {
        throw err;
      });
  }
</script>
<style>
  .w-full {
    width: 100%;
  }

  .h-full {
    height: 100%;
  }

  .instance {
    width: 250px;
  }

  .tunnel-server {
    border: none;
    height: 24px;
    line-height: 24px;
    padding: 0 7px;
    border-radius: 3px;
  }

  .dashboard-tool {
    text-align: right;
    padding: 3px 5px 5px;
    background: rgb(240 242 245);
    border-bottom: 1px solid #dfe4e8;
  }

  .label {
    display: inline-block;
    padding: 4px 2px;
  }

  .selectors-item {
    margin-right: 5px;
  }

  .btn-item {
    margin-top: 3px;
    margin-right: 5px;
  }

  #terminal {
    padding: 10px;
  }

  .xterm-screen {
    padding-left: 3px;
  }
</style>
