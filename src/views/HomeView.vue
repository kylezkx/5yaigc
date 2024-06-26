<template>
  <el-container>
    <el-header class="header">
      <h2>我的应用</h2>
    </el-header>
    <el-main class="main-content">
      <el-row :gutter="20" class="top-row">
        <el-col :span="6">
          <h1 class="centered-title">选择起点</h1>
          <el-input
            v-model="startPoint"
            placeholder="输入起点..."
            class="chat-input"
          ></el-input>
        </el-col>
        <el-col :span="6">
          <h1 class="centered-title">选择终点</h1>
          <el-input
            v-model="endPoint"
            placeholder="输入终点..."
            class="chat-input"
          ></el-input>
        </el-col>
        <el-col :span="12">
          <el-table :data="poiData" style="width: 100%">
            <el-table-column prop="name" label="名称" width="120" />
            <el-table-column prop="address" label="地址" />
            <el-table-column prop="city" label="城市" width="80" />
            <el-table-column prop="district" label="区" width="80" />
            <el-table-column prop="phone" label="电话" width="120" />
            <el-table-column prop="location" label="经纬度" width="120" />
          </el-table>
        </el-col>
      </el-row>
      <el-row :gutter="20">
        <el-col :span="8">
          <el-button type="primary" @click="sendPoiRequest" class="send-button">查询POI</el-button>
        </el-col>
        <el-col :span="16">
          <el-button type="primary" @click="loadMap" class="send-button">生成导航</el-button>
        </el-col>
      </el-row>
      <el-row :gutter="20">
        <el-col :span="12">
          <div id="map-container" class="map-container"></div>
        </el-col>
        <el-col :span="12">
          <div id="map-panel" class="map-panel"></div>
        </el-col>
      </el-row>
    </el-main>
  </el-container>
</template>

<script>
import Cookies from 'js-cookie';
import axios from "axios";
import { v4 as uuidv4 } from "uuid";
import utf8 from "utf8";
import base64 from "base-64";
import CryptoJS from "crypto-js";

export default {
  data() {
    return {
      startPoint: "", // 用户输入的起点
      endPoint: "",   // 用户输入的终点
      poiData: [],    // 存储 POI 数据
      sessionId: uuidv4(), // 唯一会话 ID
      selectedPoi: null,   // 存储选择的途径点
    };
  },
  methods: {
    ensureMapScriptLoaded() {
      return new Promise((resolve, reject) => {
        const mapScriptId = "amap-script";

        const existingScript = document.getElementById(mapScriptId);
        if (existingScript) {
          resolve(); // 地图脚本已加载
          return;
        }

        const securityScript = document.createElement("script");
        securityScript.type = "text/javascript";
        securityScript.textContent = `
          window._AMapSecurityConfig = {
            securityJsCode: "7ddbddda175331260d821e090ced568f",
          };
        `;
        document.head.appendChild(securityScript);

        const mapScript = document.createElement("script");
        mapScript.id = mapScriptId;
        mapScript.src = "https://webapi.amap.com/maps?v=2.0&key=13fa6788d2449d44f71d11f35e55d52a&plugin=AMap.Driving";
        mapScript.onload = () => {
          resolve();
        };
        mapScript.onerror = (error) => {
          reject(new Error("地图脚本加载失败"));
        };

        document.head.appendChild(mapScript);
      });
    },
    
    sendPoiRequest() {
      if (!this.startPoint.trim()) {
        console.error("起点为空");
        return;
      }

      const requestBody = {
        content: this.startPoint,
        sessionId: this.sessionId,
        queryParams: {
          city: "北京",
          page_num: 1,
          page_size: 10,
        },
      };

      const headers = this.generateAuthHeaders("POST", "/api/chat/poi-search", {});

      axios
        .post("/api/chat/poi-search", requestBody, { headers })
        .then((response) => {
          const responseData = response.data;
          if (responseData && responseData.pois) {
            this.poiData = responseData.pois; // 更新表格数据
            this.selectedPoi = responseData.pois[0]; // 选择第一个 POI 作为途径点
          } else {
            console.error("收到无效的响应:", responseData);
          }
        })
        .catch((error) => {
          console.error("请求错误:", error);
        });
    },
    
    loadMap() {
      if (!this.startPoint.trim() || !this.endPoint.trim()) {
        console.error("起点或终点为空");
        return;
      }

      if (!this.selectedPoi) {
        console.error("没有选择途径点");
        return;
      }

      this.ensureMapScriptLoaded().then(() => {
        const mapContainer = document.getElementById("map-container");
        const mapPanel = document.getElementById("map-panel");


        if (typeof AMap === "undefined") {
          console.error("地图库未加载");
          return;
        }

        const map = new AMap.Map("map-container", {
          resizeEnable: true,
          center: [116.397428, 39.90923], // 北京坐标
          zoom: 13, // 放大级别
        });

        const driving = new AMap.Driving({
          map,
          panel: mapPanel,
        });

        driving.search(
          [
            { keyword: this.startPoint, city: "北京" },
            { keyword: this.selectedPoi.name, city: this.selectedPoi.city }, // 途径点
            { keyword: this.endPoint, city: "北京" },
          ],
          (status, result) => {
            if (status === "complete") {
              console.log("路线绘制成功。");
              if (result && result.routes && result.routes.length > 0) {
                this.convertRouteToSpeech(result.routes[0]); // 将路线转换为语音
              }
            } else {
              console.error("获取路线数据失败:", result);
            }
          }
        );
      }).catch((error) => {
        console.error("地图脚本加载失败:", error);
      });
    },
    
    convertRouteToSpeech(route) {
      if (!route) {
        console.error("没有可用的路线数据");
        return;
      }

      const steps = route.steps || [];
      const routeDescription = steps.map((step) => `在${step.road}上行驶${step.distance}米，经过${step.action}`).join("。");

      const utf8Text = utf8.encode(routeDescription);
      const base64Text = base64.encode(utf8Text);

      const wsUri = "wss://api-ai.vivo.com.cn/tts"; // WebSocket 语音合成服务
      const TokenKey = Cookies.get("Admin-Token"); // 获取 Authorization 令牌

      const ws = new WebSocket(wsUri, [TokenKey]); // 传递令牌作为协议参数

      ws.onopen = () => {
        const requestBody = {
          text: base64Text,
          vcn: "vivoHelper",
          speed: 50,
          volume: 50,
          aue: 0, // PCM
          auf: "audio/L16;rate=24000", // 音频格式
          sfl: 1, // 输出格式
        };
        ws.send(JSON.stringify(requestBody));
      };

      ws.onmessage = (event) => {
        const response = JSON.parse(event.data);
        if (response.error_code !== 0) {
          console.error("TTS 错误:", response.error_msg);
        } else {
          const audioData = response.data.audio;
          const audioBlob = new Blob([base64.toUint8Array(audioData)], { type: "audio/wav" });
          const audioUrl = URL.createObjectURL(audioBlob);

          const audio = new Audio(audioUrl); // 创建音频对象
          audio.play(); // 播放音频
        }
      };

      ws.onclose = () => {
        console.log("WebSocket 连接关闭。");
      };

      ws.onerror = (error) => {
        console.error("WebSocket 错误:", error);
      };
    },
    
    generateAuthHeaders(method, uri, queryParams) {
      const appId = "3039325692"; // 应用 ID
      const appKey = "mLxeEcDxzUkngSoo"; // 应用密钥

      const timestamp = Math.floor(Date.now() / 1000).toString(); // 时间戳
      const nonce = uuidv4().substring(0, 8); // 随机数
      
      const signedHeaders = `x-ai-gateway-app-id;x-ai-gateway-timestamp;x-ai-gateway-nonce`;
      
      const canonicalQueryString = this.genCanonicalQueryString(queryParams); // 规范化查询字符串
      
      const signingString = [
        method,
        uri,
        canonicalQueryString,
        appId,
        timestamp,
        `x-ai-gateway-app-id:${appId}\nx-ai-gateway-timestamp:${timestamp}\nx-ai-gateway-nonce:${nonce}`,
      ].join("\n"); // 签名字符串
      
      const signature = CryptoJS.HmacSHA256(signingString, appKey).toString(CryptoJS.enc.Hex); // HMAC-SHA256
      
      return {
        "x-ai-gateway-app-id": appId,
        "x-ai-gateway-timestamp": timestamp,
        "x-ai-gateway-nonce": nonce,
        "x-ai-gateway-signedheaders": signedHeaders,
        "x-ai-gateway-signature": signature,
      };
    },
    
    genCanonicalQueryString(queryParams) {
      const queryString = Object.keys(queryParams)
        .sort() // 按键排序
        .map((key) => `${encodeURIComponent(key)}=${encodeURIComponent(queryParams[key])}`) // 编码键和值
        .join("&"); // 使用 "&" 连接

      return queryString;
    },
  },
  mounted() {
    this.ensureMapScriptLoaded()
      .then(() => {
        console.log("地图脚本加载成功");
      })
      .catch((error) => {
        console.error("地图脚本加载失败:", error);
      });
  },
};
</script>

<style scoped>
.header {
  background-color: #f5f5f5;
  padding: 10px;
}

.main-content {
  padding: 10px;
}

.centered-title {
  text-align: center;
}

.chat-input {
  width: 100%;
}

.send-button {
  margin-top: 10px;
}

.map-container {
  height: 300px;
}

.map-panel {
  border: 1px solid #ccc;
  padding: 10px;
}
</style>
