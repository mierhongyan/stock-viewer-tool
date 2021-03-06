<template>
  <div id="box">
    <!--顶栏部分-->
    <div class="header">
      <h1>自选小工具</h1>
      <div class="action-buttons">
        <a class="button-pin" @click="addOptional">
          <i class="icon-add"></i>
        </a>
        <a class="button-min" @click="setWinMin">
          <i class="icon-min"></i>
        </a>
        <a class="button-close" @click="setWinClose">
          <i class="icon-close"></i>
        </a>
      </div>
    </div>
    <!--指数部分-->
    <div class="stock-index">
      <div v-for="(item,index) in indexs"
           :key="index"
           @contextmenu="showContext(item)"
           @dblclick="showDetail(item)"
           :class="comparePrice(item.gain.percent,0)">
        <div class="upper-info">
          <h3>{{item.name}}</h3>
          <p>{{item.price.toFixed(2)}}</p>
        </div>
        <p class="index-gain">
          <span>{{item.gain.price>0?'+':''}}{{item.gain.price.toFixed(2)}}</span>
          <span>{{item.gain.percent>0?'+':''}}{{item.gain.percent.toFixed(2)}}%</span>
        </p>
      </div>
    </div>
    <!--自选部分-->
    <div class="table-container">
      <el-table class="optional-stock-table"
                height="500"
                size="small"
                row-key="code"
                :data="optionals"
                :header-cell-style="{padding:0}"
                @row-contextmenu="showContext"
                @row-dblclick="showDetail">
        <el-table-column label="股票" width="100">
          <template slot-scope="props">
            <div class="stock-info">
              <div :title="props.row.name">
                <h3>{{props.row.name}}</h3>
                <span v-if="props.row.status">
                  {{props.row.status==='S'?'停':'退'}}
                </span>
              </div>
              <p>{{props.row.code.toUpperCase()}}</p>
            </div>
          </template>
        </el-table-column>
        <el-table-column prop="price" label="最新价" align="right" width="75" sortable>
          <template slot-scope="props">
            <p class="stock-price">{{props.row.price.toFixed(2)}}</p>
          </template>
        </el-table-column>
        <el-table-column label="成交量" align="right" width="90">
          <template slot-scope="props">
            <p class="stock-volume">
              {{transVolume(props.row.volume)}}{{props.row.code.indexOf('hk')>-1 ?
              '股':'手'}}
            </p>
          </template>
        </el-table-column>
        <el-table-column prop="gain.percent" label="涨跌幅" align="right" sortable>
          <template slot-scope="props">
            <template v-if="!props.row.status">
            <span class="gain-price"
                  :class="comparePrice(props.row.gain.percent,0)">
              {{props.row.gain.price>0?'+':''}}{{props.row.gain.price.toFixed(2)}}
            </span>
              <span class="gain-percent"
                    :class="comparePrice(props.row.gain.percent,0)">
              {{props.row.gain.percent>0?'+':''}}{{props.row.gain.percent.toFixed(2)}}%
            </span>
            </template>
            <template v-else>
              <span class="gain-price">-</span>
              <span class="gain-percent">0.00%</span>
            </template>
          </template>
        </el-table-column>
      </el-table>
    </div>
    <!--加自选弹窗-->
    <optional-dialog ref="optionalDialog" @finish="fetchData"/>
  </div>
</template>

<script>
  import {apiUrl, timeSpan, stockIndex} from '../libs/constant' // api,请求间隔,三大指数
  import Sortable from 'sortablejs' // 表格拖拽库
  import OptionalDialog from '../components/optional' // 添加功能组件

  const shell = require('electron').shell
  const ipc = window.require('electron').ipcRenderer
  const Mousetrap = require('mousetrap') // 键位映射库
  const drag = require('electron-drag') // 窗体移动库

  export default {
    components: {
      OptionalDialog
    },
    data () {
      return {
        indexs: [], // 指数部分
        optionals: [] // 自选部分
      }
    },
    created () {
      this.fetchData()
    },
    mounted () {
      setInterval(this.fetchData, timeSpan)// 轮训获取数据
      this.rowDrop() // 拖拽功能
      this.bindKey() // 键盘映射
      this.bindMouse() // 鼠标移动
      this.bindDrag() // 绑定顶栏拖拽
      this.initIpcListener() // 初始化主窗口监听事件
    },
    methods: {
      /**
       * 通过腾讯接口获取指数&自选股的数据
       */
      fetchData () {
        let index = stockIndex.map(item => 's_' + item) // 接口需要加前缀
        let storage = localStorage.getItem('optionals')
        let storageOptional = []
        if (storage !== '' && storage !== null) {
          storageOptional = storage.split(',')
        }
        let optionals = storageOptional.map(item => 's_' + item.toLowerCase())
        let all = index.concat(optionals)
        let query = all.join(',')
        this.$http.get(apiUrl, {params: {q: query}}).then(res => {
          this.resolveData(res.data)
        })
      },
      /**
       * 解析所有获取的股票数据
       * @param data {string} - 股票数据字符串
       */
      resolveData (data) {
        this.indexs = []
        this.optionals = []
        data = data.replace(/[\r\n]/g, '') // 去掉回车换行
        data = data.substring(0, data.length - 1) // 删除最后一位字符
        let splitData = data.split(';')
        splitData.forEach((item, index) => {
          let splitItem = item.split('=') // 按照等于号截取前后部分
          let prefix = splitItem[0] // 前缀部分 // 等号前
          let suffix = splitItem[1].match(/"(.*?)"/)[1] // 后缀内容，只取双引号内容
          let content = suffix.substring(0, suffix.length - 1).split('~')
          let stock = {
            code: prefix.substring(4, 12),
            name: content[1],
            price: parseFloat(content[3]),
            gain: {
              price: parseFloat(content[4]),
              percent: parseFloat(content[5])
            },
            volume: content[6],
            status: content[8] === '' ? null : content[8]
          }
          if (index < stockIndex.length) {
            this.indexs.push(stock)
          } else {
            this.optionals.push(stock)
          }
        })
        this.updateToolTips(this.optionals)
      },

      /**
       * 弹窗添加自选股
       */
      addOptional () {
        this.$refs['optionalDialog'].show()
      },
      /**
       * 删除自选
       * @param code {string} - 个股代码
       */
      deleteStock (code) {
        let storage = localStorage.getItem('optionals')
        let storageOptional = storage.split(',')
        let final = []
        storageOptional.forEach(item => {
          if (item.toLowerCase() !== code.toLowerCase()) {
            final.push(item)
          }
        })
        localStorage.setItem('optionals', final.join(','))
        this.fetchData()
        this.$toasted.show('已删除', {
          theme: 'toasted-primary',
          position: 'bottom-center',
          duration: 2000
        })
      },
      /**
       * 执行修改数据
       */
      modifyData () {
        const modified = this.optionals.map((item) => item.code)
        localStorage.setItem('optionals', modified.join(','))
      },
      /**
       * 显示单支股票的详情，通过新窗口显示
       * @param row {object} - 个股数据
       */
      showDetail (row) {
        ipc.send('create', row.code)
      },
      /**
       * 自选股的行拖拽排序功能
       */
      rowDrop () {
        const tbody = document.querySelector('.el-table__body-wrapper tbody')
        const _this = this
        Sortable.create(tbody, {
          animation: 120,
          onEnd ({newIndex, oldIndex}) {
            const currRow = _this.optionals.splice(oldIndex, 1)[0]
            _this.optionals.splice(newIndex, 0, currRow)
            _this.modifyData()
          }
        })
      },

      /**
       * 点击跳转雪球网站
       * @param code {string} - 个股代码
       */
      openInXueQiu (code) {
        let marketPrefix = code.substring(0, 2).toLowerCase()
        shell.openExternal('https://xueqiu.com/S/' + (marketPrefix === 'hk' ? code.substring(2) : code))
      },
      /**
       * 点击跳转东方财富网站
       * * @param code {string} - 个股代码
       */
      openInGuBa (code) {
        let marketPrefix = code.substring(0, 2).toLowerCase()
        shell.openExternal('http://guba.eastmoney.com/list,' + (marketPrefix === 'hk' ? code : (code === 'sh000001' ? 'szzs' : code.substring(2))) + '.html')
      },

      /**
       * 主窗口线程事件监听
       */
      initIpcListener () {
        ipc.on('show-xueqiu', (event, code) => {
          this.openInXueQiu(code)
        })
        ipc.on('show-guba', (event, code) => {
          this.openInGuBa(code)
        })
        ipc.on('place-top', (event, code) => {
          let storage = localStorage.getItem('optionals')
          let storageOptional = storage.split(',')
          if (storageOptional.indexOf(code) > -1) {
            storageOptional.splice(storageOptional.indexOf(code), 1)
          }
          localStorage.setItem('optionals', code + ',' + storageOptional.join(','))
          this.fetchData()
        })
        ipc.on('delete-stock', (event, code) => {
          this.deleteStock(code)
        })
      },
      /**
       * 关闭主窗口
       */
      setWinClose () {
        ipc.send('main-window-close')
      },
      /**
       * 最小化主窗口
       */
      setWinMin () {
        ipc.send('main-window-min')
      },
      /**
       * 更新系统托盘的提示文字，显示当前前几条个股的信息
       * @param data {array} - 所有自选股的数据
       * @description windows下只能显示部分文字，会溢出，待解决
       */
      updateToolTips (data) {
        let content = data.map(item => {
          return `${item.name}：当前价 ¥${item.price},涨跌 ¥${item.gain.price}（${item.gain.percent}%）`
        })
        ipc.send('main-tray-update', content.join('\n'))
      },
      /**
       * 右键指数&个股显示上下文菜单
       * @param row {object} - 个股数据
       */
      showContext (row) {
        ipc.send('main-right-click', row.code)
      },

      /**
       * 绑定键盘映射
       */
      bindKey () {
        Mousetrap.bind(['0', '1', '2', '3', '4', '5', '6', '7', '8', '9', 'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n', 'o', 'p', 'q', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z'], (e) => {
          this.$refs['optionalDialog'].show(e.key)
        })
        Mousetrap.bind(['command+k', 'ctrl+k'], function () {
          ipc.send('open-devtools')
        })
      },
      /**
       * 使用 electron-drag 解决顶栏无法捕获鼠标事件的问题（暂不支持linux）
       */
      bindDrag () {
        drag('.header')
        // 如果不支持则使用样式的方式
        if (!drag.supported) {
          document.querySelector('.header').style['-webkit-app-region'] = 'drag'
        }
      },
      /**
       * 单独为页面做事件监听可以防止出现部分处于 body 中的 element ui 元素监听不到的问题
       */
      bindMouse () {
        document.addEventListener('mouseenter', this.mouseEnter)
        document.addEventListener('mouseleave', this.mouseLeave)
      },
      /**
       * 鼠标进入主窗口的情况
       */
      mouseEnter () {
        ipc.send('main-mouse-enter')
      },
      /**
       * 鼠标移出主窗口的情况
       */
      mouseLeave () {
        ipc.send('main-mouse-leave')
      }
    }
  }
</script>

<style lang="scss">
  @import '../scss/index';
</style>
