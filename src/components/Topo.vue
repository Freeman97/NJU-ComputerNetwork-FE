<template>
  <el-container style="height: 100vh;">
    <!-- 页眉 -->
    <el-header class="header-container">
      <h1 class="title">动态路由</h1>
    </el-header>
    <el-container style="height: 100%">
      <!-- 侧边节点库 -->
      <!-- <el-aside class="aside"> -->
        <!-- 侧栏菜单 -->
        <!-- <el-menu> -->
          <!-- 子菜单 -->
          <!-- <el-submenu v-for="(type, index) in typeList" :key="type" :index="String(index)">
            <template slot="title">
              <i class="el-icon-s-grid"></i>
              <span>{{type}}</span>
            </template>
            <el-menu-item-group>
              <el-menu-item
              v-for="item in libraryList[type]"
              :key="item.id"
              class="library-item">
                <div draggable="true" @dragstart="dragToBoardStart">
                  <img :src="item.pic" :alt="item.name" draggable="false">
                  <span>{{item.name}}</span>
                </div>
              </el-menu-item>
            </el-menu-item-group>
          </el-submenu> -->
        <!-- </el-menu> -->
      <!-- </el-aside> -->
      <!-- 拓扑画板 -->
      <el-main class="board-container">
        <!-- topo中的节点的右键菜单 -->
        <Context-Menu :position="position" v-if="showMenu" @menuClick="clickMenuItem" />
        <!-- 功能按键 -->
        <div class="button-container">
          <!-- <el-button @click="saveTopo">保存拓扑</el-button>
          <el-button @click="clearTopo" type="danger">清空拓扑</el-button> -->
          <el-button @click="sendPacket">控制路由器B发送数据包</el-button>
        </div>
        <!-- 画板 -->
        <svg 
        class="board" 
        ondragover="return false" 
        @drop="dropToBoard" 
        oncontextmenu="return false"
        @click.left.stop.prevent="closeContextMenu">
          <!-- 已连接的线 -->
          <line
          v-for="(item, index) in lines"
          :key="index"
          :x1="item.x1" :y1="item.y1"
          :x2="item.x2" :y2="item.y2"
          style="stroke:rgb(255,0,0);stroke-width:2"/>
          <!-- 正在连接的线 -->
          <line
          :x1="connecting.x1" :y1="connecting.y1"
          :x2="connecting.x2" :y2="connecting.y2"
          style="stroke:rgb(255,0,0);stroke-width:2"/>
          <!-- topo图上的节点 -->
          <g 
          v-for="(item, index) in topoNodes"
          :key="item.id"
          @mousedown.left.stop.prevent="moveAndLink(index, $event)"
          @click.right.stop.prevent="nodeMenu(index, $event)">
            <image :xlink:href="item.pic" width="50px" height="50px" :x="item.x" :y="item.y"></image>
            <text :x="item.x + 25" :y="item.y + 66" style="text-anchor: middle; user-select: none;">{{item.name}}</text>
          </g>
        </svg>
      </el-main>
      <el-dialog
        title="设置接口信息"
        :visible.sync="setIntVisible"
        width="540px"
        @close="closeSetIntDialog">
        <el-form ref="form" :model="form" label-width="100px"> 
          <el-form-item label="接口类型">
              <el-radio-group v-model="interfaceType">
                  <el-radio label="f" name="interfaceType">快速以太网</el-radio>
                  <el-radio label="s" name="interfaceType">串行接口</el-radio>
              </el-radio-group>
          </el-form-item>
          <el-form-item label="第几个接口">
              <el-radio-group v-model="interfaceId">
                  <el-radio label="0" name="interfaceId">0</el-radio>
                  <el-radio label="1" name="interfaceId">1</el-radio>
              </el-radio-group>
          </el-form-item>
          <el-form-item label="ipAddress">
              <el-input v-model="form.ipAddress" placeholder="请输入ipAddress"></el-input>
          </el-form-item>
          <el-form-item label="subnetMask">
              <el-input v-model="form.subnetMask" placeholder="请输入subnetMask"></el-input>
          </el-form-item>
          <el-button type="primary" style="width:100%; height:40px;" @click="onSetInt">提交</el-button>
        </el-form>
      </el-dialog>

      <el-dialog        
        title="查看接口信息"
        :visible.sync="checkIntVisible"
        width="540px"
        @close="closeCheckIntDialog">
        <div align="right">
          <el-button icon="el-icon-search" style="height:40px;" @click="oncheckInt">查看</el-button>
        </div>
        <el-form ref="form" :model="checkIntForm" label-width="100px"> 
          <el-form-item label="接口类型">
              <el-radio-group v-model="checkIntForm.interfaceType">
                  <el-radio label="f" name="interfaceType">快速以太网</el-radio>
                  <el-radio label="s" name="interfaceType">串行接口</el-radio>
              </el-radio-group>
          </el-form-item>
          <el-form-item label="第几个接口">
              <el-radio-group v-model="checkIntForm.interfaceId">
                  <el-radio label="0" name="interfaceId">0</el-radio>
                  <el-radio label="1" name="interfaceId">1</el-radio>
              </el-radio-group>
          </el-form-item>
        </el-form>
        <el-card shadow="hover">
          IP地址: {{ intInfo.ipAddress }}, 子网掩码: {{ intInfo.subnetMask }}, 是否启用: {{ intInfo.status }}
        </el-card>
      </el-dialog>
      <el-dialog        
        title="路由表信息"
        :visible.sync="checkRouterVisible"
        width="700px">
        <el-card shadow="hover">
          <div style="white-space: pre-wrap;" v-html="wrapLineRouterInfo"></div>
        </el-card>
      </el-dialog>
      <!-- 启动RIP -->
      <el-dialog title="启动RIP动态路由" :visible.sync="enableRIPVisible" width="540px">
        <el-table :data="intListInfo" style="width:100%">
          <el-table-column prop="interface" label="接口"></el-table-column>
          <el-table-column prop="ipAddress" label="IP地址"></el-table-column>
          <el-table-column prop="subnetMask" label="子网掩码"></el-table-column>
          <el-table-column prop="status" label="是否已启用"></el-table-column>
          <el-table-column label="开启RIP动态路由">
            <template slot-scope="scope">
              <el-button @click="handleEnableRIP(scope.row)" type="text">启动</el-button>
            </template>
          </el-table-column>
        </el-table>
        <el-form ref="form" :model="enableRIPForm" label-width="100px">
          <el-form-item></el-form-item>
        </el-form>
      </el-dialog>
      <el-dialog
      :visible.sync="checkPacketVisible"
      width="1000px">
        <div style="" v-html="packetInfo"></div>
      </el-dialog>
    </el-container>
    <!-- <el-footer height="60px" style="border-top: 1px solid #CCC;">
      <a href="https://github.com/laddwong" class="address">项目地址</a>
    </el-footer> -->
  </el-container>

</template>
<script>
/* eslint-disable */
import ContextMenu from './ContextMenu'
import { MessageBox } from 'element-ui'
import nodeData from '../data/nodeData'
import defaultNodeData from '../data/defaultNodeData'
import defaultLinkData from '../data/defaultLinkData'
export default {
  name: 'Topo',
  components: {
    ContextMenu
  },
  data () {
    return {
      libraryList: {}, // 左侧节点库的节点数据
      typeList: [], // 节点分类
      topoNodes: defaultNodeData, // topo图中的节点
      topoLinks: defaultLinkData, // topo图中的连线
      connecting:{ // 显示正在连接的线条
        x1: 0,
        y1: 0,
        x2: 0,
        y2: 0
      },
      move: true, // 操作模式，默认为移动。可切换为连接模式
      position: {x: 0, y: 0}, // 右键菜单的位置
      showMenu: false, // 控制右键菜单的显示与否
      indexOfMenu: null, // 表示在哪个节点上点击了右键菜单
      setIntVisible: false,
      form: {
          ipAddress: "",
          subnetMask: ""
      },
      router: '',
      interfaceType: '',
      interfaceId: '',
      // 命令数组
      cmdArr: [
          {
              id: 1,
              title: "配置单个路由器上指定接口的IP地址和子网掩码",
              key: "router"
          },
      ],
      // 选中命令
      activeItem: "",
      // 返回数据
      resData: "",
      checkIntForm: {
          interfaceType: '',
          interfaceId: ''
      },
      enableRIPForm: {

      },
      checkIntVisible: false,
      intInfo: '',
      intListInfo: [],
      routerInfo: '',
      checkRouterVisible: false,
      enableRIPVisible: false,
      checkPacketVisible: false,
      packetInfo: ''
    }
  },
  methods: {
    // 从左边的节点库拖出节点
    dragToBoardStart (e) {
      // 设置拖出的数据
      e.dataTransfer.setData('text/plain', JSON.stringify({pic: e.target.children[0].src, name: e.target.children[1].innerText}))
      e.dataTransfer.effectAllowed = "copy" // 设置拖的操作为复制操作
      // window.console.log(e)
    },
    // 节点拖放到topo图区域，即新建节点
    dropToBoard (e) {
      const content = JSON.parse(e.dataTransfer.getData('text/plain')) // 接收来自拖出的内容,并还原为对象
      const date = new Date()
      let node = {
        id: date.getTime(), // 用时间戳生成唯一id，Symbol类型的id不能存储到本地
        pic: content.pic, // 图片
        name: content.name, // 默认显示名称，可修改
        x: e.layerX, // 横坐标
        y: e.layerY, // 纵坐标
      }
      this.topoNodes.push(node)
    },
    // 移动topo图中的节点，连接节点
    moveAndLink (index, e) {
      // 判断当前模式
      if (this.move) {
        // 移动模式
        const layerX = e.layerX - this.topoNodes[index].x
        const layerY = e.layerY - this.topoNodes[index].y
        document.onmousemove = (e) => {
          this.topoNodes[index].x = e.layerX - layerX 
          this.topoNodes[index].y = e.layerY - layerY
        }
        document.onmouseup = () => {
          document.onmousemove = null
          document.onmouseup = null
        }
      } else {
        // 连线模式
        this.topoLinks.push({
          startNodeId: this.topoNodes[this.indexOfMenu].id,
          endNodeId: this.topoNodes[index].id,
          startInterface: 'fa0/1',
          endInterface: 'fa0/1'
        })
        this.connecting = { // 重置正在连接的线
          x1: 0,
          y1: 0,
          x2: 0,
          y2: 0
        }
        document.onmousemove = null // 重置鼠标移动事件
        this.move = true // 重置为移动模式
      }
    },
    // 显示topo图上的节点的右键菜单
    nodeMenu (index, e) {
      this.position = {x: e.offsetX, y: e.offsetY}
      this.indexOfMenu = index
      if(this.topoNodes[this.indexOfMenu].name != 'Switch' && this.topoNodes[this.indexOfMenu].name != 'Server') {
        this.showMenu = true
      }
    },
    // 执行右键菜单的功能
    clickMenuItem (option) {
      // 关闭右键菜单
      this.showMenu = false
      // 连接功能
      if (option === 'link') {
        // 设置为连线模式
        this.move = false
        // 创建连线
        this.connecting = {
          x1: this.topoNodes[this.indexOfMenu].x + 20,
          y1: this.topoNodes[this.indexOfMenu].y + 20,
          x2: this.topoNodes[this.indexOfMenu].x + 20,
          y2: this.topoNodes[this.indexOfMenu].y + 20
        }
        // 连线终点跟随鼠标
        document.onmousemove = (e) => {
          this.connecting.x2 = e.layerX
          this.connecting.y2 = e.layerY
        }
      }
      // 重命名功能
      if (option === 'rename') {
        MessageBox.prompt('请输入新名称', '重命名', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          inputPattern: /\S/,
          inputErrorMessage: '名称不能为空'
        }).then(({ value }) => {
          this.topoNodes[this.indexOfMenu].name = value
        })
      }
      //设置接口信息
      if (option === 'ip') {
          this.router = this.topoNodes[this.indexOfMenu].name
          this.setIntVisible = true;
        // MessageBox.prompt('请输入ip地址', '地址信息', {
        //   confirmButtonText: '确定',
        //   cancelButtonText: '取消',
        //   inputPattern: /\S/,
        //   inputErrorMessage: '不能为空'
        // }).then(({ value }) => {
        //   this.topoNodes[this.indexOfMenu].name = value
        // })
      }
      //查看接口信息
      if (option === 'checkInt') {
        this.router = this.topoNodes[this.indexOfMenu].name
        this.checkIntVisible = true
      }
      //查看路由表信息
      if (option === 'checkRouter') {
        this.router = this.topoNodes[this.indexOfMenu].name
        var _this = this;
          this.$axios
              .get("http://127.0.0.1:8000/api/router/" + this.router)
              .then(res => {
                  this.resData = JSON.stringify(res.data);
                  _this.routerInfo = res.data;
                  console.log(this.resData);
              });        
        this.checkRouterVisible = true
      }
      // 开启RIP
      if (option === 'enableRIP') {
        this.router = this.topoNodes[this.indexOfMenu].name
        var _this = this
        this.$axios
          .get("http://127.0.0.1:8000/api/router/" + this.router + "/interfaces")
          .then(res => {
            _this.intListInfo = res.data
            console.log(res)
          })
        this.enableRIPVisible = true
      }
      // 删除功能
      if (option === 'delete') {
        MessageBox.confirm(`是否删除节点 "${this.topoNodes[this.indexOfMenu].name}"`, '删除节点', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.topoNodes.splice(this.indexOfMenu, 1)
        })
      }
    },
    // 点击空白地方，关闭右键菜单，取消连线模式
    closeContextMenu () {
      // 关闭右键菜单
      this.position = {x: 0, y: 0}
      this.showMenu = false
      this.indexOfMenu = null
      // 取消连线模式
      this.move = true
      this.connecting = {
        x1: 0,
        y1: 0,
        x2: 0,
        y2: 0
      }
      document.onmousemove = null
    },
    // 保存Topo
    saveTopo () {
      localStorage.topoNodes = JSON.stringify(this.topoNodes)
      localStorage.topoLinks = JSON.stringify(this.topoLinks)
      MessageBox('保存成功')
    },
    clearTopo () {
      MessageBox.confirm('是否清空当前拓扑图？', '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        localStorage.removeItem('topoNodes')
        localStorage.removeItem('topoLinks')
        this.topoNodes = []
        this.topoLinks = []
      })
    },
    onSetInt() {
      var _this = this;
        this.$axios
            .patch("http://127.0.0.1:8000/api/router/" + this.router + '/' + this.interfaceType + '/' + this.interfaceId, {
              "ipAddress": _this.form.ipAddress,
              "subnetMask": _this.form.subnetMask
            })
            .then(res => {
                this.resData = JSON.stringify(res.data);
                console.log(this.resData);
            });
        _this.$message({
          message: '接口已配置成功！',
          type: 'success'
        });
    },
    oncheckInt() {
      //console.log(this.router)
      //console.log("http://127.0.0.1:8000/api/router/" + this.router + '/' + this.checkIntForm.interfaceType + '/' + this.checkIntForm.interfaceId)
      var _this = this;
        this.$axios
            .get("http://127.0.0.1:8000/api/router/" + this.router + '/' + this.checkIntForm.interfaceType + '/' + this.checkIntForm.interfaceId)
            .then(res => {
                _this.$message({
                  message: '接口信息查询成功！',
                  type: 'success'
                });
                this.resData = JSON.stringify(res.data);
                _this.intInfo = res.data;
                console.log(this.resData);
            });
    },
    handleEnableRIP(row) {
      var _this = this
      console.log("subnet is :  " + _this.calculateSubnet(row.ipAddress, row.subnetMask))
      this.$axios
        .patch("http://127.0.0.1:8000/api/router/" + this.router, { netList: [_this.calculateSubnet(row.ipAddress, row.subnetMask)] })
        .then(res => {
          _this.$message({
            message: 'RIP开启成功！',
            type: 'success'
          });
          console.log(res)
        })
    },
    calculateSubnet(ip, subnetMask) {
      var splitIP = ip.split(".")
      var splitSubnetMask = subnetMask.split(".")
      var res = ''
      for(var i = 0; i < 3; i++) {
        res += (parseInt(splitIP[i]) & parseInt(splitSubnetMask[i]))
        res += '.'
      }
      res += (parseInt(splitIP[3]) & parseInt(splitSubnetMask[3]))
      return res
    },
    // 关闭接口设置对话框清空表单
    closeSetIntDialog(){
      this.interfaceType = ''
      this.interfaceId = ''
      this.form.ipAddress = ''
      this.form.subnetMask = ''
    },
    closeCheckIntDialog() {
      this.intInfo = ''
    },
    sendPacket() {
      var _this = this;
      this.$axios
        .post("http://127.0.0.1:8000/api/router/B/ippacket", {
          ipAddress: "10.0.0.1",
          filterSubnet: "10.0.0.0"
        }).then(res => {
          _this.$message({
            message: '发送成功！',
            type: 'success'
          });
          // this.text = res.data.replace(/\n/g, '<br>')
          _this.packetInfo = res.data.replace(/\n/g, '<br>')
          _this.checkPacketVisible = true
          console.log(res.data)
        })
    }
    },
  computed: {
    // 动态计算节点间的连线
    lines () {
      let hash = {}
      const OFFSET = 20
      this.topoNodes.forEach((node, index) => {
        hash[node.id] = index
      })
      return this.topoLinks.map(item => {
        const startNode = this.topoNodes[hash[item.startNodeId]]
        const endNode = this.topoNodes[hash[item.endNodeId]]
        return {
          x1: startNode.x + OFFSET,
          y1: startNode.y + OFFSET,
          x2: endNode.x + OFFSET,
          y2: endNode.y + OFFSET,
          startInterface: item.startInterface,
          endInterface: item.endInterface
        }
      })
    },
    wrapLineRouterInfo() {
      console.log(this.routerInfo)
      return this.routerInfo.replace(/\\r\\n/g, "<br/>")
    }
  },
  mounted () {
    this.libraryList = nodeData
    this.typeList = Object.keys(this.libraryList)
    if (localStorage.topoNodes && localStorage.topoLinks) {
      this.topoNodes = JSON.parse(localStorage.topoNodes)
      this.topoLinks = JSON.parse(localStorage.topoLinks)
    }
  }
}
</script>

<style scoped>

.aside {
  background-color: #F2F6FC;
  border-right: 2px solid #F2F6FC;
  height: 100%;
}
.library-item {
  color: #606266;
}
.library-item img {
  width: 30px;
  height: 30px;
  margin-right: 10px;
}
.board-container {
  padding: 0;
  position: relative;
}
.button-container {
  position: absolute;
  right: 16px;
  top: 10px;
}
.board {
  background-color: #FFF;
  height: 99%;
  width: 100%;
}
.header-container {
  background-color: #409EFF;
}
.title {
  margin: 0;
  line-height: 60px;
  color: #FFF;
}
.address {
  color: #409EFF;
  line-height: 60px;
}
</style>
