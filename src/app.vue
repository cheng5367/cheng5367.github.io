<template>
  <div id="app">
    <right-nav :version="APP_VERSION" :settings="settings" :show-menu.sync="showMenu" :mode="mode"></right-nav>
    <!-- <normal-nav :version="APP_VERSION" :settings="settings" :show-menu="showMenu"></normal-nav> -->

    <div id="buttons">
      <el-button size="mini" @click="getYaolingInfo">妖灵</el-button>
      <el-button size="mini" @click="filterDialogVisible = true">自定义筛选</el-button>
      <el-button size="mini" @click="getToken">添加token</el-button>
      <el-button size="mini" @click="getLeiTal">擂台搜索</el-button>
      <div v-if="mode === 'wide'">
        <div style="font-size: 14px;">
          <div>当前线程数: {{sockets.length}}/{{thread}}</div>
          <template v-for="(socket, index) in sockets" >
            <p :key="index" v-if="socket">线程.{{index+1}} {{socket.task ? `正在执行任务.${socket.task.taskIndex}` : '空闲'}}</p>
          </template>
          <div v-if="radarTask">任务进度:{{closedTask}}/{{radarTask.tasks.length}}</div>
        </div>
      </div>
    </div>
    <!--页面主要布局-->
    <div class="main">
      <div class="left"><div id="qmap"></div></div>
      <div class="right">
        <ul>
          <li v-for="(item,index) in getSearchYaoLing" :key="index">
            <span class="name">{{getSerYlName(item.sprite_id)}}</span>
            <span class="gps">{{item.latitude | latFile}},{{item.longtitude}}</span>
          </li>
        </ul>
      </div>
    </div>
    <!--页面主要布局-->

    <radar-progress :show="progressShow" :max-range="max_range" :thread="thread" :percent="progressPercent"></radar-progress>
    <el-dialog
    title="自定义筛选"
    :visible.sync="filterDialogVisible"
    class="filter-dialog">
    <div class="check">
      <el-checkbox v-model="settings.use_custom">启用</el-checkbox>
    </div>
    <el-row :gutter="10" class="filter-list">
      <el-col v-for="(yl, index) in settings.custom_filter" :key="index" :xs="12" :sm="8" :md="6" :lg="4" :xl="3">
        <div class="filter-content" :class="{active : yl.on}" @click="yl.on = !yl.on">
          <img :src="'https://hy.gwgo.qq.com/sync/pet/'+yl.img" :alt="yl.name">
          <span :class="{active:up(yl.id)}">{{ yl.name }}</span>
        </div>
      </el-col>
    </el-row>
  </el-dialog>


  <el-dialog
  title="输入token"
  :visible.sync="tokenEle"
  width="50%">
  <el-input v-model="openidVal" placeholder="openid"></el-input>
  <el-input v-model="gwgoTokenVal" placeholder="gwgo_token"></el-input>
  <span slot="footer" class="dialog-footer">
    <el-button @click="tokenEle = false">取 消</el-button>
    <el-button type="primary" @click="tokenTrue">确 定</el-button>
  </span>
</el-dialog>

<!--擂台控件布局-->
<div class="ui-canvas" v-if="isLeiTai"></div>
<div class="ui-leitai" v-if="isLeiTai">
  <div class="ui-leitai-cont">
    <i class="el-icon-error" @click="closeCanvas"></i>
    <div class="ui-header">
      <label>经度:</label>
      <input value="" class="ui-lt-input" v-model="ltLat"/>
      <label>纬度:</label>
      <input value="" class="ui-lt-input" v-model="ltLong"/>
      <span class="ui-lt-btn" @click="serLeiTai">查询</span>
    </div>
    <ul class="leitaiInfo-lisit">
      <li>
        <div class="lt-Name">擂主名称</div>
        <div class="lt-lat">经度</div>
        <div class="lt-long">纬度</div>
        <div class="lt-zl">胜利者战力</div>
        <div class="lt-yl">胜利者妖灵</div>
        <div class="lt-opt">操作</div>
      </li>
      <li v-for="(item,index) in leiTaiInfo" :key="index" v-if="item.state == 0">
        <div class="lt-Name">{{item.winner_name}}</div>
        <div class="lt-lat">{{item.latitude}}</div>
        <div class="lt-long">{{item.longtitude}}</div>
        <div class="lt-zl">{{item.winner_fightpower}}</div>
        <div class="lt-yl">
          <el-popover
            placement="right"
            width="300"
            trigger="click">
            <el-table :data="item.sprite_list">
              <el-table-column width="150" property="spriteid" label="名称" :formatter="formatter"></el-table-column>
              <el-table-column width="100" property="fightpower" label="战力"></el-table-column>
              <el-table-column width="50" property="level" label="等级"></el-table-column>
            </el-table>
            <el-button slot="reference">查询擂台妖灵信息</el-button>
          </el-popover>
        </div>
        <div class="lt-opt">
          <el-button slot="reference" disabled>复制</el-button>
        </div>
      </li>
    </ul>
  </div>
</div>
<!--擂台控件布局-->
  </div>
</template>
<script>
import tempdata from './lib/tempdata';
import activities from './lib/activities';
import mixins from './mixins/mixins';
import bot from './mixins/bot';
import map from './mixins/map';
import RightNav from './components/rightNav';
import socket from './mixins/socket';
import RadarProgress from './components/radarProgress';
import RadarTasks from './lib/RadarTasks';
import { getLocalStorage, setLocalStorage } from './lib/util';
import {
  FILTER,
  API_KEY,
  CUR_YAOLING_VERSION,
  APP_VERSION,
  WIDE_SEARCH,
  userInfo
} from './lib/config';

export default {
  name: 'zhuoyao-radar',
  mixins: [mixins, bot, map, socket],
  components: {
    RightNav,
    RadarProgress
  },
  data() {
    let showMenu = false;
    let location = getLocalStorage('radar_location');
    let settings = getLocalStorage('radar_settings');
    let defaultSettings = {
      fit: {
        t1: false,
        t2: false,
        all: false,
        nest: false,
        rare: true,
        fish: false,
        feature: false,
        element: false,
      },
      auto_search: true,
      show_time: true,
      position_sync: false,
      wide: FILTER.FILTER_WIDE,
      custom_filter:FILTER.FILTER_CUSTOM,
      use_custom:false,
      show_box:false,
      version:APP_VERSION,
    };

    let flag = false;
    let ans = [];

    //如果第一次打开网页 则打开菜单
    if (!settings) {
      showMenu = true;
    } else {
      //对于settings.custom_filter
      //在版本更新后availableYaolings发生变动时，custom_filter会保留以前的键
      //因此需要在此处做一个remapping

      if (settings.custom_filter) {
        let sc = settings.custom_filter;
        let scMap = [];
        sc.forEach(o => {
          scMap[o.id] = o;
        });
        defaultSettings.custom_filter.forEach((v,i,a) => {
          if (scMap.hasOwnProperty(v.id)) ans.push(scMap[v.id]);
          else ans.push(v);
        });
        flag = true;
      }
    }

    settings = Object.assign({}, defaultSettings, settings || {});

    if (flag) {
      settings.custom_filter = ans;
    }


    if (!(location && settings.position_sync)) {
      location = {
        longitude: 116.3579177856,
        latitude: 39.9610780334,
        zoom:16,
      };
    }

    let range = Number(this.$parent.range || WIDE_SEARCH.MAX_RANGE);
    let max_range = range * 2 + 1; // 经纬方向单元格最大数
    // 线程数最多6个
    let thread = Math.min(
      Number(this.$parent.thread || WIDE_SEARCH.MAX_SOCKETS),
      6
    );
    if (this.$parent.mode === 'temp') {
      range = 0;
      thread = 1;
    }
    return {
      location,
      settings,
      showMenu,
      APP_VERSION,
      range,
      thread,
      max_range,
      mode: this.$parent.mode,
      sockets: new Array(thread),
      radarTask: null,
      searching: false,
      map: {},
      clickMarker: null, // 点击位置标记
      userMarker: null, // 用户位置标记
      searchBoxMarker: [], //大范围搜索框标记
      searchBoxWideSet: new Set(), //大范围搜索集合
      searchOutboxMarker: null, //外围指引框标记
      currVersion: CUR_YAOLING_VERSION, //190508版本的json 如果有变动手动更新
      yaolings: tempdata.Data,
      upYaolings: activities.Data,
      markers: new Map(), //妖灵markerclass
      messageMap: new Map(), // 缓存请求类型和id
      botMode: false,
      botInterval: null,
      botTime: 0,
      botGroup: '群号',
      botChecked: [],
      botWelcomeInfo: '捉妖扫描机器人2.1启动~有什么问题可以@我哦',
      botLocation: {
        longitude: 116.3579177856,
        latitude: 39.9610780334
      },
      progressShow: false,
      filterDialogVisible:false,
      tokenEle: false,
      openidVal: '',
      gwgoTokenVal: '',
      getSearchYaoLing: [],
      isLeiTai: false, // 擂台组件控制开关
      leiTaiInfo: [],
      // 擂台经纬度
      ltLong: '',
      ltLat: '',
    };
  },
  mounted() {
    // 初始化地图组件
    this.initMap();

    // 初始化websocket
    this.initSockets();
    // 获取妖灵

    this.getSettingFileName();

    // 获取用户位置
    if (!this.settings.position_sync) {
      this.getLocation()
        .then(
          position => {
            this.location.longitude = position.longitude;
            this.location.latitude = position.latitude;

            var pos = new qq.maps.LatLng(
              this.location.latitude,
              this.location.longitude
            );
            this.map.panTo(pos);
            this.userMarker = new qq.maps.Marker({
              position: pos,
              map: this.map
            });
          },
          e => {
            console.log(e);
            if (e.code === 3) {
              // this.notify('无法获取设备位置信息');
            }
          }
        )
        .catch(b => {});
    }

    if (this.mode === 'wide') {
      this.notify(
        `大范围搜索开启，当前最大搜索单位:${Math.pow(
          this.max_range,
          2
        )}个.线程数:${this.thread}个.`
      );
    }


    // 增加键盘事件
    // document.onkeydown = function (e) {
    //   let key = window.event.keyCode;
    //   if (key == 123 || key == 17 || key == 83) {
    //     window.close();
    //   }
    // };

  },
  filters: {
    latFile: function(latVal){
      return latVal.toString().slice(0,2)+ ','+latVal.toString().slice(2,latVal.length);
    },
    longFile: function(longVal){
      return longVal.toString().slice(0,2)+ ','+longVal.toString().slice(2,longVal.length);
    },
  },
  methods: {
    subLat: function(latVal) {
      var a = latVal.substring(0,2) + "," + latVal.substring(2,latVal.length);
      return a;
    },
    getSerchYl: function(yl){
      this.getSearchYaoLing.push(yl);
    },

    getSerYlName: function(ylId){
      for(var i = 0;i < this.yaolings.length;i+=1){
        if(this.yaolings[i].Id === ylId){
          return this.yaolings[i].Name;
        }
      }
    },
    /**
     * 跨域获取最新妖灵数据
     */
    getYaolings: function() {
      var url = 'https://hy.gwgo.qq.com/sync/pet/config/' + this.currVersion;
      // $.ajax({
      //   url: url,
      //   type: 'post',
      //   // 用于设置响应体的类型 注意 跟 data 参数没关系！！！
      //   dataType: 'jsonp',
      //   success: function (res) {
      //     alert("aaa");
      //     console.log(res)
      //   }
      // })
      try {
          $.getJSON(url,result => {
              console.log(result);
              this.statusOK = true;
          });
          console.log("1");
      } catch(e) {
          console.log(e);
          console.log("3");
      }
      console.log("2");
    },
    /**
     * 根据查询结果过滤数据，打标记
     */
    buildMarkersByData: function(t) {
      if (t && t.length) {
        t.forEach(item => {
          if (
            this.fit[0] === 'special' ||
            this.fit.indexOf(item.sprite_id) > -1
          ) {
            this.addMarkers(item);
          }
        });
      }
    },
    /**
     * 添加token
    */
    getToken: function(){
      this.tokenEle = true;
    },
    /**
     * 根据妖灵ID获取妖灵名称
    */
    getYaolingName: function(ylId) {
      for(var i =0;i< this.yaolings.length;i++) {
        if (ylId == this.yaolings[i].Id) {
          return this.yaolings[i].Name;
        }
      }
    },
    tokenTrue: function(){
      this.tokenEle = false;
      this.$cookies.set("openid", this.openidVal)
      this.$cookies.set("gwgo_token", this.gwgoTokenVal)
    },
    /**
     * 获取妖灵数据
     */
    getYaolingInfo: function() {
      if (this.botMode) return;

      // 先清除标记
      this.clearAllMarkers();

      if (this.mode === 'normal') {
        if (this.searchOutboxMarker != null) this.searchOutboxMarker.setMap(null);
        if (this.searchBoxMarker.length > 0) {
          this.searchBoxMarker[0].setMap(null);
          this.searchBoxMarker.pop();
        }

        this.buildSearchboxMarker(this.location.latitude,this.location.longitude,true);
        this.sendMessage(this.initSocketMessage('1001'));
      } else {
        if (this.searching) {
          this.notify("请等待此次搜索结束！");
          return false;
        }

        this.clearAllBox();

        this.radarTask = new RadarTasks({
          range: this.range,
          lng: this.location.longitude,
          lat: this.location.latitude
        });


        if(this.mode === 'wide') this.progressShow = true;
        this.lng_count = this.lat_count = 0;
        this.searching = true;
        for (let index = 0; index < WIDE_SEARCH.MAX_SOCKETS; index++) {
          let socket = this.sockets[index];
          if (socket) {
            this.startTaskWithSocket(socket);
          }
        }
      }
    },
    /**
     * 获取擂台数据
     */
    getLeitaiInfo: function(data) {
      this.leiTaiInfo = data;
      console.log(this.yaolings);
    },
    /**
     * 获取官方配置文件
     */
    getSettingFileName: function() {
      this.sendMessage(this.initSocketMessage('10041'));
    },
    /**
     * 暂未使用
     */
    getBossLevelConfig: function() {
      return;
      this.sendMessage(this.initSocketMessage('10040'));
    },
    /**
     * 是否为活动up妖灵
     */
    up:function(val) {
      return this.upYaolings.hasOwnProperty(val) && this.upYaolings[val] ;
    },

    /**
     * 擂台信息
    */
    getLeiTal: function(){
      this.isLeiTai = true;
      this.sendMessage(this.initSocketMessage('1002',{
        longitude: 116.3919281960,
        latitude: 39.8704202963
      }));
    },
    serLeiTai: function() {
      var T = this;
      // ltLong ltLat绝对不能为空
      if(this.ltLong != "" && this.ltLat != "") {
        this.leiTaiInfo = [];
        T.sendMessage(this.initSocketMessage('1002',{
          longitude: Number(T.ltLong),
          latitude: Number(T.ltLat)
        }));
      }
    },
    formatter(row,column){
      for(var i = 0;i<this.yaolings.length;i++) {
        if(this.yaolings[i].Id == row.spriteid) {
          return this.yaolings[i].Name;
        }
      }
    },
    closeCanvas: function() {
      this.isLeiTai = false;
      this.leiTaiInfo = [];
    },

  },
  computed: {
    fit: function() {
      let ans = [];

      //自定义筛选优先级最高
      if (this.settings.use_custom) {
        return Array.from(new Set(
          this.settings.custom_filter
          .filter(item => item.on)
          .map(item => item.id)
        ));
      }

      if (this.mode === 'normal') {

        let _fit = this.settings.fit;
        if (_fit.all) {
          return ['special'];
        }


        // 根据值把key转换成FILTER_FISH这种，取常量配置中的值
        for (let _f in _fit) {
          if (_fit[_f]) {
            let _arr = FILTER[`FILTER_${_f.toLocaleUpperCase()}`];
            ans = ans.concat(_arr);
          }
        }
        return Array.from(new Set(ans));
      } else {
        let _fit = this.settings.wide
          .filter(item => item.on)
          .map(item => item.id);
        return Array.from(new Set(_fit));
      }
    },
    /**
     * 大范围进度条
     */
    progressPercent: function() {
      let result = 0;
      if (this.radarTask) {
        let tasks = this.radarTask.tasks;
        let _close = tasks.filter(i => {
          return i.status === 'close';
        }).length;
        result = Math.floor(_close / tasks.length * 100);
      }
      // let _open = tasks.length
      return result;
    },
    /**
     * 已完成任务数
     */
    closedTask: function() {
      let result = 0;
      if (this.radarTask) {
        result = this.radarTask.tasks.filter(i => {
          return i.status === 'close';
        }).length;
      }
      return result;
    },
  },
  watch: {
    settings: {
      handler: function(newV, oldV) {
        console.log('settings update...');
        setLocalStorage('radar_settings', this.settings);
      },
      deep: true
    }
  }
};
</script>
<style lang='less'>
#app {
  // padding-left: 200px;
  position: relative;
  height: 100%;
}
.main{
  width: 100%;
  height: 100%;
}
.left{
  width: 70%;
  height: 100%;
  float: left;
}
.right{
  width: 30%;
  height: 100%;
  float: right;
  ul{
    height: 800px;
    padding-top: 50px;
    padding-left: 15px;
    overflow-y:scroll;
    li{
      margin-bottom: 20px;
      span{
        display: inline-block;
        margin-right: 10px;
      }
      .name{
        width: 100px;
        text-align: left;
      }
      .gps{
        width: 175px;
      }
      .time{
        width: 100px;
      }
    }
  }
}
ul,li{
  list-style-type:none
}
.ui-canvas{
  position: fixed;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  background-color: #000;
  z-index: 100;
  opacity: .5;
}
.ui-leitai{
  position: fixed;
  top: 0;
  left: 50%;
  width: 1000px;
  height: 700px;
  margin-left: -500px;
  background-color: #fff;
  z-index: 101;
  border-radius: 10px;
  padding: 20px;
  &-cont{
    position: relative;
    width: 100%;
    height: 100%;
    .el-icon-error{
      position: absolute;
      right: 20px;
      top: 0;
      font-size: 25px;
      cursor: pointer;
    }
    .leitaiInfo-lisit{
      height: 600px;
      overflow: hidden;
      overflow-y: scroll;
      li{
        margin-bottom: 15px;
        overflow: hidden;
        div{
          float: left;
        }
        .lt-Name{
          width: 200px;
        }
        .lt-lat{
          width: 150px;
        }
        .lt-long{
          width: 150px;
        }
        .lt-zl{
          width: 150px;
        }
        .lt-yl{
          width: 200px;
        }
        .lt-opt{
          width: 100px;
        }
      }
    }
  }
}

.ui-header{
  margin-bottom: 20px;
}
.ui-lt-input{
  width: 150px;
  -webkit-appearance: none;
  background-color: #fff;
  background-image: none;
  border-radius: 4px;
  border: 1px solid #dcdfe6;
  box-sizing: border-box;
  color: #606266;
  display: inline-block;
  font-size: inherit;
  height: 30px;
  line-height: 30px;
  outline: none;
  padding: 0 15px;
  transition: border-color .2s cubic-bezier(.645,.045,.355,1);
}
.ui-lt-btn{
  display: inline-block;
    line-height: 1;
    white-space: nowrap;
    cursor: pointer;
    background: #fff;
    border: 1px solid #dcdfe6;
    color: #606266;
    text-align: center;
    box-sizing: border-box;
    margin: 0;
    transition: .1s;
    font-weight: 500;
    -moz-user-select: none;
    -webkit-user-select: none;
    -ms-user-select: none;
    padding: 12px 20px;
    font-size: 14px;
    border-radius: 4px;
    color: #fff;
    background-color: #409eff;
    border-color: #409eff;
    padding: 10px 20px;
    font-size: 14px;
    border-radius: 4px;
}
</style>

