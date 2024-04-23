<template>
  <el-container>
    <el-header class="header"></el-header>
    <el-main class="main-content">
      <el-row :gutter="20" class="top-row">
        <el-col :span="8">
          <h1 class="centered-title">5y去哪玩</h1>
          <el-input
            type="textarea"
            v-model="message.content"
            placeholder="在这里输入你的关键词..."
            class="chat-input"
          ></el-input>
          <el-button type="primary" @click="sendMessage" class="send-button">发送</el-button>
        </el-col>
        <el-col :span="16">
          <el-table :data="poiData" style="width: 100%">
            <el-table-column prop="name" label="名称" width="120" />
            <el-table-column prop="address" label="地址" />
            <el-table-column prop="city" label="城市" width="80" />
            <el-table-column prop="district" label="区" width="80" />
            <el-table-column prop="phone" label="电话" width="120" />
            <el-table-column prop="location" label="经纬度" width="120" />
            <el-table-column prop="distance" label="距离" width="80" />
          </el-table>
        </el-col>
      </el-row>
      <el-row :gutter="20">
        <el-col :span="12">
          <div id="container"></div>
        </el-col>
        <el-col :span="12">
          <div id="panel"></div>
        </el-col>
      </el-row>
    </el-main>
  </el-container>
</template>



<script>
import axios from 'axios';
import { v4 as uuidv4 } from 'uuid';

axios.defaults.baseURL = 'http://localhost:28080'; // 根据你的后端服务器地址进行设置

export default {
  data() {
    return {
      message: {
        content: '',
        queryParams: {
          city: '北京',
          page_num: '1',
          page_size: '10'
        }
      },
      poiData: [], // 用于存储搜索到的 POI 数据
      sessionId: uuidv4(),
      selectedPois: [] // 用于存储后端返回的前两个 POI 数据
    };
  },
  methods: {
    sendMessage() {
      if (!this.message.content.trim()) {
        return; // 阻止发送空消息
      }

      const requestBody = {
        content: this.message.content,
        sessionId: this.sessionId,
        queryParams: this.message.queryParams
      };

      axios
        .post('/api/chat/poi-search', requestBody)
        .then((response) => {
          const responseData = response.data;
          if (responseData && responseData.pois) {
            this.poiData = responseData.pois; // 更新表格数据
            this.selectedPois = responseData.pois.slice(0, 2); // 存储前两个 POI 数据
            this.loadMap(); // 在这里调用 loadMap
          } else {
            console.error('收到错误或格式无效:', responseData);
          }
        })
        .catch((error) => {
          console.error('请求错误:', error);
        });

      this.message.content = ''; // 清空输入框
    },
    loadMap() {
      // 确保 selectedPois[0] 和 selectedPois[1] 都存在
      if (!this.selectedPois[0] || !this.selectedPois[1]) {
        console.error('没有足够的 POI 数据来显示地图');
        return;
      }

      // 设置安全密钥
      const securityScript = document.createElement('script');
      securityScript.type = 'text/javascript';
      securityScript.text = `
        window._AMapSecurityConfig = {
          securityJsCode: "7ddbddda175331260d821e090ced568f",
        };
      `;
      document.head.appendChild(securityScript);

      // 加载高德地图的 JS API
      const mapScript = document.createElement('script');
      mapScript.type = 'text/javascript';
      mapScript.src = 'https://webapi.amap.com/maps?v=2.0&key=13fa6788d2449d44f71d11f35e55d52a&plugin=AMap.Driving';
      mapScript.onload = () => {
        // 初始化地图
        var map = new AMap.Map("container", {
          resizeEnable: true,
          center: [116.397428, 39.90923],//地图中心点
          zoom: 13 //地图显示的缩放级别
        });
        //构造路线导航类
        var driving = new AMap.Driving({
          map: map,
          panel: "panel"
        }); 
        // 根据起终点名称规划驾车导航路线
        driving.search([
          {keyword: this.selectedPois[0].name, city: this.selectedPois[0].city},
          {keyword: this.selectedPois[1].name, city: this.selectedPois[1].city}
        ], function(status, result) {
          // result 即是对应的驾车导航信息，相关数据结构文档请参考  https://lbs.amap.com/api/javascript-api/reference/route-search#m_DrivingResult
          if (status === 'complete') {
            console.log('绘制驾车路线完成')
          } else {
            console.error('获取驾车数据失败：' + result)
          }
        });
      };
      document.body.appendChild(mapScript);
    }
  }
};
</script>

<style scoped>
.header {
  text-align: left;
}

.main-content {
  padding-top: 0;
}

.centered-title {
  text-align: center;
  color: #3eaf7c;
  font-size: 2em;
  margin-bottom: 0.5em;
}

.send-button {
  margin-top: 1em;
  background-color: #3eaf7c;
  color: white;
}

.chat-input {
  font-size: 1.2em;
  font-weight: bold;
}

.el-table {
  margin-top: 1em;
}

.el-table th {
  background-color: #f5f5f5;
  color: #333;
}

.el-table td {
  color: #666;
}

#container {
  width: 100%;
  height: 60vh; /* 增加高度 */
}

#panel {
  width: 100%;
  height: 60vh; /* 增加高度 */
  overflow-y: auto;
  background-color: white;
}

#panel .amap-call {
  background-color: #009cf9;
  border-top-left-radius: 4px;
  border-top-right-radius: 4px;
}

#panel .amap-lib-driving {
  border-bottom-left-radius: 4px;
  border-bottom-right-radius: 4px;
  overflow: hidden;
}

.top-row {
  margin-top: -20px;
}
</style>