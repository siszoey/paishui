<template>
  <div class="panel-container">
    <div class="op-box">
      <div class="item-head" style="margin-top:0">查询图层</div>
      <el-form label-width="70px">
        <el-form-item label="选取图层" style="margin:0">
          <el-select v-model="selectLayer" value-key="value" placeholder="请选择图层" style="width:100%" size="small" >
            <el-option v-for="item in datasetOptions" :key="item.value" :label="item.label" :value="item"> </el-option>
          </el-select>
        </el-form-item>
      </el-form>
    </div>
    <!-- 绘制方式 -->
    <div class="op-box">
      <div class="item-head" style="margin-top:0">查询范围</div>
      <el-radio v-model="drawType" label="full">全图</el-radio>
      <el-radio v-model="drawType" label="extent">绘制范围</el-radio>
    </div>
    <div class="op-box">
      <el-button type="primary" size="small" style="width:100%" @click="doQuery" :loading="queryStatus">开始查询</el-button>
    </div>
    <!-- 查询结果 -->
    <div class="query-result">
      <el-row>
        <el-col :span="24">
          <div class="item-head">查询结果</div>
        </el-col>
        <el-col :span="24">
          <el-table :data="resultData" v-loading="queryStatus"
                    :header-cell-style="{fontSize: '14px', fontWeight:'600',background:'#eaf1fd',color:'#909399'}" style="width: 100%">
            <el-table-column prop="layer" align="center" label="图层"></el-table-column>
            <el-table-column prop="number" align="center" label="数量(个)"></el-table-column>
            <el-table-column align="center" label="操作">
              <template slot-scope="scope">
                <el-button type="text" @click="showQueryResultData(scope.row.data)">查看</el-button>
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
import { GeoJSON } from 'ol/format';
import { createBox } from 'ol/interaction/Draw';
import * as turf from '@turf/turf'
import { fromCircle as circleToPolygon } from 'ol/geom/Polygon';
import iDraw from '@/views/zhpt/common/mapUtil/draw'
import iQuery from '@/views/zhpt/common/mapUtil/query'
import { comSymbol } from '@/utils/comSymbol';
import { fieldDoc, pointFieldDoc } from '@/views/zhpt/common/doc'
import { Polygon, LineString, Point } from 'ol/geom';
import { Feature } from 'ol';

export default {
  props: ['data'],
  data() {
    return {
      selectLayer: '',
      
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
      const dataSetInfo = appconfig.gisResource['iserver_resource'].dataService.dataSetInfo
      return dataSetInfo.map(item => {
        return { label: item.label, value: item.name, type: item.type }
      })
    }
  },
  watch: {
    // 监听面板是否被改变
    '$store.state.map.P_editableTabsValue': function (val, oldVal) {
        if (val !== "facilitySearch") {
            this.removeAll()
        } else {
            this.init()
        }
    },
    drawType(val, oldVal) {
      if (val === 'extent') this.initDraw()
      else {
          this.drawFeature = null
          this.lightLayer.getSource().clear()
          this.drawer && this.drawer.end()
      }
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
    removeAll() {
      this.drawer && this.drawer.end()
      this.vectorLayer && this.mapView.removeLayer(this.vectorLayer)
      this.lightLayer && this.mapView.removeLayer(this.lightLayer)
      this.drawer = this.vectorLayer = this.lightLayer = null
      this.$store.dispatch('map/handelClose', {
        box:'HalfPanel',
        pathId: 'queryResultMore',
        widgetid: 'HalfPanel',
      });
    },
    init() {
      this.vectorLayer = new VectorLayer({ source: new VectorSource(), style: comSymbol.getAllStyle(3, '#f40', 5, '#f00') })
      this.lightLayer = new VectorLayer({ source: new VectorSource(), style: comSymbol.getAllStyle(3, '#00FFFF', 5, '#00FFFF') })
      this.mapView.addLayer(this.vectorLayer)
      this.mapView.addLayer(this.lightLayer)
    },
    /**
    * 初始化绘制组件
    */
    initDraw () {
      this.drawer && this.drawer.end()
      this.vectorLayer && this.vectorLayer.getSource().clear()
      this.drawer = new iDraw(this.data.mapView, 'polygon', {
        endDrawCallBack: feature => {
          this.drawer.remove()
          this.drawFeature = feature
        },
        showCloser: false
      })
      this.drawer.start()
    },

    doQuery () {
        if (!this.selectLayer) return this.$message.error('请先选择要分析的图层!')
        if (this.drawType === "") return this.$message.error('请选择查询范围')
        if (this.drawType === 'extent' && !this.drawFeature) return this.$message.error('请先绘制查询范围!')
        let dataSetInfo = [{ name: this.selectLayer.value, label: this.selectLayer.label }]

        if (!this.vectorLayer) {
            this.vectorLayer = new VectorLayer({ source: new VectorSource(), style: comSymbol.getAllStyle(3, '#f40', 5, '#00FFFF') })
            this.mapView.addLayer(this.vectorLayer)
        }
        
        let queryTask = new iQuery({ dataSetInfo })
        this.queryStatus = true
        queryTask.sqlQuery("1=1").then(resArr => {
            let resFeaturesObj = resArr.filter(res => res && res.result.featureCount !== 0)
            let tableData = []
            resFeaturesObj.forEach(obj => {
                let features = new GeoJSON().readFeatures(obj.result.features)
                // 查询图形，相交处理
                if (this.drawFeature) {
                    features  = features.filter(feature => {
                        let limitGeometry = turf.polygon(this.drawFeature.getGeometry().getCoordinates())
                        let geomtry = feature.getGeometry(), inGeometry
                        if (geomtry instanceof Point) {
                            inGeometry = turf.point(geomtry.getCoordinates())
                        } else if (geomtry instanceof LineString) {
                            inGeometry = turf.lineString(geomtry.getCoordinates())
                        } else return false
                        return turf.booleanContains(limitGeometry, inGeometry)
                    })
                }
                this.queryStatus = false
                this.vectorLayer.getSource().addFeatures(features)
                tableData.push({ name: obj.layerName, data: features.map(fea => fea.values_) })
            })
            this.addTableData(tableData)
        })
    },

    addTableData (data) {
        let doc = this.selectLayer.type === "point" ? pointFieldDoc : fieldDoc
        let keys = Object.keys(doc)
        keys.length = 15
        let fields = {}
        keys.forEach(key => {
            fields[fieldDoc[key]] = key
        })
        this.resultData = data.map(item => {
            return { 
                layer: item.name, 
                number: item.data.length,
                data: item.data,
                fields
            }
        })
    },

    beforeExport(data) {
      if (data.length == 0) this.$message.warning('无数据导出！')
    },
    clearResult() {
      this.drawFeature = null
      this.resultData = []
      this.drawType = ''
      this.selectLayer = ""
      this.drawer && this.drawer.end()
      this.vectorLayer.getSource().clear()
      this.lightLayer.getSource().clear()
    },
    /**
     *  展示查询结果
     */
    showQueryResultData(data) {
      if (data.length === 0) return 
      let doc = this.selectLayer.type === "point" ? pointFieldDoc : fieldDoc
      let colsData = []
      for (let key in doc) {
        colsData.push({ prop: key, label: doc[key]})
      }
      colsData.length = 15
      this.$store.dispatch('map/changeMethod', {
        pathId: 'queryResultMore',
        widgetid: 'HalfPanel',
        label: '详细信息',
        param: { data, colsData, rootPage: this }
      })
    },
    gotoGeometry (geometry) {
      console.log("定位")
      if (!this.lightLayer) {
        this.lightLayer = new VectorLayer({ source: new VectorSource(), style: comSymbol.getAllStyle(3, '#f40', 5, '#FFFFB6') })
        this.lightLayer.setZIndex(999)
        this.mapView.addLayer(this.lightLayer)
      }
      this.lightLayer.getSource().clear()
      this.lightFeature = new Feature({ geometry })
      this.lightLayer.getSource().addFeature(this.lightFeature)
      // 
      let extent = this.lightFeature.getGeometry().getExtent()
      let center = [(extent[0] + extent[2]) / 2, (extent[1] + extent[3]) / 2]
      this.mapView.getView().setCenter(center)
      this.mapView.getView().setZoom(20)
    }
  }
}
</script>

<style lang="scss" scoped>
@import './css.scss';
</style>