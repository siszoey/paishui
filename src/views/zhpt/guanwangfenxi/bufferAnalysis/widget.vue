<template>
  <div class="panel-container">
    <div class="op-box">
      <div class="item-head" style="margin-top:0">查询图层</div>
      <el-form label-width="70px">
        <el-form-item label="选取图层" style="margin:0">
          <el-select v-model="selectLayer" value-key="value" placeholder="请选择图层" style="width:100%" size="small" @change="selectLayerChange">
            <el-option-group v-for='group in datasetOptions' :key="group.label" :label="group.label">
               <el-option v-for="item in group.layers" :key="item.value" :label="item.label" :value="item"></el-option>
            </el-option-group>
          </el-select>
        </el-form-item>
      </el-form>
    </div>
    <!-- 绘制方式 -->
    <div class="op-box">
      <div class="item-head" style="margin-top:0">几何类型</div>
      <el-radio v-model="drawType" label="point">点</el-radio>
      <el-radio v-model="drawType" label="line">线</el-radio>
      <el-radio v-model="drawType" label="circle">圆</el-radio>
      <el-radio v-model="drawType" label="rect">矩形</el-radio>
      <el-radio v-model="drawType" label="polygon">多边形</el-radio>
    </div>
    <div class="op-box">
      <div class="item-head" style="margin-top:0">缓冲距离</div>
      <el-form label-width="auto">
        <el-form-item label="缓冲距离(米)" style="margin:0">
          <el-input-number size="small" v-model="bufferDistance" @blur="checkBufferDistance" controls-position="right" :step="10" :min="10"
                           style="width:100%">
          </el-input-number>
        </el-form-item>
      </el-form>
    </div>
    <div class="op-box">
      <el-button type="primary" size="small" style="width:100%" @click="doQuery_new" :loading="queryStatus">开始查询</el-button>
    </div>
    <!-- 查询结果 -->
    <div class="query-result">
      <el-row>
        <el-col :span="24">
          <div class="item-head">查询结果</div>
        </el-col>
        <el-col :span="24">
          <el-table :data="resultData" v-loading="queryStatus" :header-cell-style="{fontSize: '14px', fontWeight:'600',background:'#eaf1fd',color:'#909399'}" style="width: 100%">
            <template slot="empty">
              <img src="@/assets/icon/null.png" alt="">
              <p class="empty-p">暂无数据</p>
            </template>
            <el-table-column type="index" width="50" label="序号" align="center"></el-table-column>
            <el-table-column prop="layer" align="center" label="图层" ></el-table-column>
            <el-table-column prop="number" align="center" label="数量(个)"></el-table-column>
            <el-table-column align="center" label="操作">
              <template slot-scope="scope">
                <el-button type="text" @click="showQueryResultData(scope.row)">查看</el-button>
                <download-excel class="export-btn" :data="scope.row.data" :fields="scope.row.fields" type="xls" :name="scope.row.layer"
                                style="display: inline;">
                  <el-button type="text" @click="beforeExport(scope.row.data)">导出</el-button>
                </download-excel>
              </template>
            </el-table-column>
          </el-table>
        </el-col>
      </el-row>
      <el-button type="danger" style="width:100%" size="small" @click="clearResult">清除结果 </el-button>
    </div>
  </div>
</template>

<script>
import { appconfig } from 'staticPub/config'
import { SuperMap, FieldService, FeatureService } from '@supermap/iclient-ol';
import Draw from 'ol/interaction/Draw'
import { Vector as VectorSource } from 'ol/source';
import { Vector as VectorLayer } from 'ol/layer';
import { Style, Circle, Icon, Fill, RegularShape, Stroke, Text } from 'ol/style';
import { GeoJSON, WFS } from 'ol/format';
import { getFields } from '@/api/mainMap/analysis'
import { createBox } from 'ol/interaction/Draw';
import * as turf from '@turf/turf'
import { fromCircle as circleToPolygon } from 'ol/geom/Polygon';
import iDraw from '@/views/zhpt/common/mapUtil/draw'
import iQuery from '@/views/zhpt/common/mapUtil/query'
import { comSymbol } from '@/utils/comSymbol';
import { fieldDoc } from '@/views/zhpt/common/doc'
import { Feature } from 'ol';
import { mapUtil } from '../../common/mapUtil/common';

export default {
  props: ['data'],
  data() {
    return {
      selectLayer: '',
      
      bufferDistance: 50,
      draw: null,
      
      
      resultData: [],
      resFeatures: [],
      queryStatus: false,

      // 
      drawType: '',
      drawer: null,
      vectorLayer: null,
      drawFeature: null,
      lightFeature: null,
      lightLayer: null
    }
  },
  computed: {
    // 图层选项
    datasetOptions() {
      let [name, type] = appconfig.initLayers.split("&&")
      let layer = mapUtil.getAllSubLayerNames(name, type)
      // 设置图层
      return layer.sublayers.map(layer => {
        let layers = layer.sublayers.map(sub => {
          return { label: sub.title, value: sub.name.split('@')[0] }
        })
        return { label: layer.title, value: layer.name, layers }
      })
    }
  },
  watch: {
    // 监听面板是否被改变
    '$store.state.map.P_editableTabsValue': function (val, oldVal) {
      if (val !== "bufferAnalysis") this.removeAll()
      else this.init()
    },
    drawType(val, oldVal) {
      if (!val) return
      this.initDraw()
    }
  },
  destroyed() {
    this.removeAll()
  },
  mounted() {
    this.mapView = this.data.mapView
    this.init()
  },
  methods: {
    init () {
        this.vectorLayer = new VectorLayer({ source: new VectorSource(), style: mapUtil.getCommonStyle()})
        this.mapView.addLayer(this.vectorLayer)
        this.lightLayer = new VectorLayer({ source: new VectorSource(), style: mapUtil.getCommonStyle(true) })
        this.mapView.addLayer(this.lightLayer)
        this.data.that.setPopupSwitch(false)
    },
    removeAll () {
      this.drawer && this.drawer.end()
      this.vectorLayer && this.mapView.removeLayer(this.vectorLayer)
      this.lightLayer && this.mapView.removeLayer(this.lightLayer)
      this.drawer = this.vectorLayer = this.lightLayer = null
        this.data.that.setPopupSwitch(true)
      this.$store.dispatch('map/handelClose', {
        box:'HalfPanel',
        pathId: 'queryResultMore',
        widgetid: 'HalfPanel',
      });

    },
    /**
    * 初始化绘制组件
    */
    initDraw () {
      this.drawer && this.drawer.end()
      this.vectorLayer && this.vectorLayer.getSource().clear()
      this.drawer = new iDraw(this.data.mapView, this.drawType, {
        endDrawCallBack: feature => {
          // console.log(feature.getGeometry().getCoordinates())
          this.drawer.remove()
          this.drawFeature = feature
        },
        showCloser: false
      })
      this.drawer.start()
    },

    doQuery_new () {
      if (!this.selectLayer) return this.$message.error('请先选择要分析的图层!')
      if (!this.drawFeature) return this.$message.error('请先绘制缓冲区图形!')
      let dataSetInfo = [{ name: this.selectLayer.value, label: this.selectLayer.label }]
      let bufferFeature = this.getBufferFeature(this.drawFeature)
      this.vectorLayer.getSource().addFeature(bufferFeature)

      new iQuery({ dataSetInfo }).spaceQuery(bufferFeature).then(resArr => {
        console.log("空间查询信息", resArr)
        let tableData = []
        resArr.forEach(res => {
          if (res.result.featureCount !== 0) {
            let featuresJson = res.result.features
            let features = new GeoJSON().readFeatures(featuresJson)
            this.vectorLayer.getSource().addFeatures(features)
            tableData.push({ tableName: res.tableName, name: res.layerName, data: features.map(fea => fea.values_) })
          }
        })
        this.addTableData(tableData)
      })
    },

    addTableData (data) {
      let fieldsPromise = data.map(item => mapUtil.getFields(item.tableName))
      Promise.all(fieldsPromise).then(fieldsArr => {
        this.resultData = data.map((item, index) => {
          let fields = {}
          fieldsArr[index].forEach(item => {
            fields[item.name] = item.field
          })
          return {
            tableName: item.tableName,
            layer: item.name,
            number: item.data.length,
            data: item.data,
            fields
          }
        })
      })

    },

    checkBufferDistance() {
      if (!this.bufferDistance) { this.bufferDistance = 50 }
    },
    /**
     * 选择图层
     */
    selectLayerChange() {
      // 
    },
    /**
     * 要素缓冲
     */
    getBufferFeature(drawFeature) {
      let geometry = drawFeature.getGeometry()
      let fea = null
      if (this.drawType == 'point') { fea = turf.point(geometry.getCoordinates()) } // turf的点要素
      else if (this.drawType == 'line') { fea = turf.lineString(geometry.getCoordinates()) } // turf的线要素
      else if (this.drawType == 'circle') { fea = turf.polygon(circleToPolygon(geometry).getCoordinates()) }
      else { fea = turf.polygon(geometry.getCoordinates()) } // turf的面要素
      let resultFeature = new GeoJSON().readFeature(turf.buffer(fea, this.bufferDistance / 1000, { units: 'kilometers' }))
      resultFeature.setStyle(new Style({
        stroke: new Stroke({
          color: '#0099ff',
          width: 3
        }),
        fill: new Fill({
          color: 'rgba(255, 255, 255,0.6)'
        }),
      }))
      // this.vectorLayer.getSource().addFeature(resultFeature)
      return resultFeature
    },
    beforeExport(data) {
      if (data.length == 0) this.$message.warning('无数据导出！')
    },
    clearResult() {
      this.vectorLayer.getSource().clear()
      this.drawFeature = null
      this.resultData = []
      this.drawType = ''
      this.selectLayer = ''
    },
    /**
     *  展示查询结果
     */
    showQueryResultData(row) {
      mapUtil.getFields(row.tableName).then(res => {
        let colsData = res.map(item => {
          return { prop: item.field, label: item.name }
        })
        let data = row.data
        this.$store.dispatch('map/changeMethod', {
          pathId: 'queryResultMore',
          widgetid: 'HalfPanel',
          label: '详细信息',
          param: { data, colsData, rootPage: this }
        })
      })


      
    },
    gotoGeometry (geometry) {
      this.lightLayer.getSource().clear()
      this.lightFeature = new Feature({ geometry })
      let center = mapUtil.getCenter(this.lightFeature)
      this.lightLayer.getSource().addFeature(this.lightFeature)
      new mapUtil(this.mapView).setZoomAndCenter(20, center)
    }
  }
}
</script>

<style lang="scss" scoped>
@import './css.scss';
</style>