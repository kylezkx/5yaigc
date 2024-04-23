<template>
    <div style="display: flex; height: 100vh;">
        <div id="container"></div>
        <div id="panel"></div>
    </div>
</template>

<script>
export default {
    name: "MapContainer",
    mounted() {
        this.$nextTick(() => {
            this.loadMap();
        });
    },
    methods: {
        loadMap() {
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
                    {keyword: '北京市地震局(公交站)',city:'北京'},
                    {keyword: '亦庄文化园(地铁站)',city:'北京'}
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
#container {
  width: 50%;
  height: 100%;
}
#panel {
  width: 50%;
  height: 100%;
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
</style>