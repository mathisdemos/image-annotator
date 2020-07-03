<template>
  <div>
    <div id="map" class="map" style="border: 2px solid black"></div>
<!--    <p>Current interaction: {{ this.currentInteraction }}</p>-->
<!--    <button @click="toggleInteraction">Toggle interaction</button>-->
    <div v-if="showLabelInput" >
      <input v-model="labelText"/>
      <button @click="setLabel">Set label</button>
    </div>
  </div>
</template>

<script>
import 'ol/ol.css';
import Map from 'ol/Map';
import ol from 'ol'

// static image
import View from 'ol/View';
import {getCenter} from 'ol/extent';
import ImageLayer from 'ol/layer/Image';
import Projection from 'ol/proj/Projection';
import Static from 'ol/source/ImageStatic';
import GeoJSON from 'ol/format/GeoJSON'

// drawing
import {Draw, Select, Snap, Modify, defaults as defaultInteractions} from 'ol/interaction';
import {createBox} from 'ol/interaction/Draw';
import {Vector as VectorLayer} from 'ol/layer';
import {Vector as VectorSource} from 'ol/source';
import {Circle as CircleStyle, Fill, Stroke, Style, Text} from 'ol/style';
import Feature from 'ol/Feature';
import Polygon from 'ol/geom/Polygon';
import Collection from 'ol/Collection';
import ol_interaction_Transform from 'ol-ext/interaction/Transform';
import throttle from 'lodash.throttle'

export default {
  name: 'ImageAnnotator',

  data () {
    return {
      map: null,
      currentInteraction: 'draw',
      currentFeature: null,
      labelText: '',
      showLabelInput: false,
      currentFeatures: {}
    }
  },

  methods: {
    setLabel () {
      const style = new Style({
        fill: new Fill({
          color: 'rgba(255, 255, 255, 0.2)'
        }),
        stroke: new Stroke({
          color: '#ffcc33',
          width: 2
        }),
        image: new CircleStyle({
          radius: 7,
          fill: new Fill({
            color: '#ffcc33'
          })
        }),
        text: new Text({
          text: this.labelText,
          font: '20px Arial',
          placement: 'line',
          maxAngle: 0,
          textBaseline: 'top',
          overflow: true,
        })
      })
      this.currentFeature.setStyle(style)

      this.labelText = ''
      this.currentFeature = null
      this.showLabelInput = false
    },

    initMap() {
      // Map views always need a projection.  Here we just want to map image
      // coordinates directly to map coordinates, so we create a projection that uses
      // the image extent in pixels.
      const extent = [0, 0, 2479, 3508];
      const projection = new Projection({
        code: 'image',
        units: 'pixels',
        extent: extent
      });

      const testGeoJSON = {'type': 'FeatureCollection',
        'features': [{'type': 'Feature',
          'id': '1',
          'geometry': {'type': 'Polygon',
            'coordinates': [[[231, 2542],
              [2222, 2542],
              [2222, 1494],
              [231, 1494],
              [231, 2542]]]},
          'properties': {'score': 0.9999485015869141, 'label': 'Section'}},
          {'type': 'Feature',
            'id': '2',
            'geometry': {'type': 'Polygon',
              'coordinates': [[[232, 3054],
                [2192, 3054],
                [2192, 2633],
                [232, 2633],
                [232, 3054]]]},
            'properties': {'score': 0.999942421913147, 'label': 'Section'}},
          {'type': 'Feature',
            'id': '3',
            'geometry': {'type': 'Polygon',
              'coordinates': [[[270, 1221],
                [2219, 1221],
                [2219, 809],
                [270, 809],
                [270, 1221]]]},
            'properties': {'score': 0.9999086856842041, 'label': 'Section'}},
          {'type': 'Feature',
            'id': '4',
            'geometry': {'type': 'Polygon',
              'coordinates': [[[1065, 114],
                [1416, 114],
                [1416, 46],
                [1065, 46],
                [1065, 114]]]},
            'properties': {'score': 0.9965907335281372, 'label': 'Page numbers'}},
          {'type': 'Feature',
            'id': '5',
            'geometry': {'type': 'Polygon',
              'coordinates': [[[238, 1339],
                [1264, 1339],
                [1264, 1251],
                [238, 1251],
                [238, 1339]]]},
            'properties': {'score': 0.9962388277053833, 'label': 'Crossheading'}}]}

      const boxLayerSource = new VectorSource({
        features: (new GeoJSON()).readFeatures(testGeoJSON),
        format: new GeoJSON()
      })
      const boxLayer = new VectorLayer({
        source: boxLayerSource,
        style: new Style({
          fill: new Fill({
            color: 'rgba(255, 255, 255, 0.2)'
          }),
          stroke: new Stroke({
            color: '#ffcc33',
            width: 2
          }),
          image: new CircleStyle({
            radius: 7,
            fill: new Fill({
              color: '#ffcc33'
            })
          }),
          // text: new Text({
          //   text: 'abcde',
          //   font: '20px Arial',
          //   placement: 'line',
          //   maxAngle: 0,
          //   textBaseline: 'top',
          //   overflow: true,
          // })
        })
      });

      this.vectorLayer = boxLayer

      // boxLayer.events.register('vertexmodified', this, function(vertex, feature, pixel) {
      //   //do something with the vertex
      //   console.log('vertex modified')
      // });

      this.map = new Map({
        interactions: defaultInteractions({
          doubleClickZoom: false,
          dragAndDrop: false,
          dragPan: false,
          keyboardPan: false,
          keyboardZoom: false,
          mouseWheelZoom: false,
          pointer: false,
          select: true
        }),
        layers: [
          new ImageLayer({
            source: new Static({
              url: '/test_image.png',
              projection: projection,
              imageExtent: extent
            })
          }), boxLayer
        ],
        target: 'map',
        view: new View({
          projection: projection,
          center: getCenter(extent),
          zoom: 2,
          maxZoom: 8
        })
      });

      // MODIFIY INTERACTION
      this.modifyInteraction = new Modify({source: boxLayerSource, insertVertexCondition: () => false});

      this.modifyInteraction.on('modifystart', (e) => {
        console.log('modify start')
/*        console.log(e.mapBrowserEvent.target)
        console.log(e.features)*/
        this.currentFeatures = e.features.getArray().reduce((acc, val) => {
          return {
            ...acc,
            [val.getId()]: {
              revision: val.getRevision(),
              initialCoords: JSON.parse(JSON.stringify(val.getGeometry().getCoordinates()[0]))
            }
          }
        }, {})
/*        const feature = e.features.getArray()[0]
        const modified = e.features.getArray().reduce((acc, val) => {
          console.log(val.getRevision())
          return val.getRevision() > 1 || acc
        }, false)
        console.log(modified)
        const initCoordinates = feature.getGeometry().getCoordinates()[0]
        console.log('init', initCoordinates)
        console.log(feature)

        feature.on('change', (e) => {
          const newCoords = e.target.getGeometry().getCoordinates()[0]
          console.log('new', newCoords)

          initCoordinates.forEach((point, idx) => {
            if (!this.currentVertex || this.currentVertex === -1) {
              this.currentVertex = newCoords.find((coordArray, idx) => {
                return coordArray[0] !== point[0] || coordArray[1] !== point[1]
              })
              //console.log(this.currentVertex)
            }
          })*/
        // });
      })

        // function modifySiblingCorners (e) {
        //   const newCoords = e.target.getGeometry().getCoordinates()[0]
        //   console.log(this.initCoordinates)
        //   console.log('changed', newCoords)
        //   // const extent = geometry.getExtent()
        //   // const newCoords = [[
        //   //   [extent[0], extent[1]],
        //   //   [extent[2], extent[1]],
        //   //   [extent[2], extent[3]],
        //   //   [extent[0], extent[3]],
        //   //   [extent[0], extent[1]],
        //   // ]]
        //
        //   // feature.un('change', modifySiblingCorners);
        //   // geometry.setCoordinates(newCoords, 'XY')
        //   // // Reenabling change event
        //   // feature.on('change', modifySiblingCorners);
        // }

        this.modifyInteraction.on('modifyend', (e) => {
          console.log('modify end')
          const modified = e.features.getArray().reduce((acc, val) => {
            const oldFeatureRevision = this.currentFeatures[val.getId()].revision
            return oldFeatureRevision !== val.getRevision() ? val : acc
          }, {})
          const newCoordinates = modified.getGeometry().getCoordinates()[0]
          const oldCoordinates = this.currentFeatures[modified.getId()].initialCoords
          const changed = newCoordinates.reduce((acc, val, idx) => {
            let out = acc
            const isChanged = val[0] !== oldCoordinates[idx][0] || val[1] !== oldCoordinates[idx][1]
            if (isChanged) {
              out.idx = idx
              out.newCoords = val
              out.oldCoords = oldCoordinates[idx]
            }
            return out
          }, {})
          console.log(changed)
          const findMaxChange = (changed) => {
            let out = {}
            let firstDiff = Math.abs(changed.newCoords[0] - changed.oldCoords[0])
            let secondDiff = Math.abs(changed.newCoords[1] - changed.oldCoords[1])
            if (firstDiff > secondDiff) {
              out.old = changed.oldCoords[0]
              out.coord = changed.newCoords[0]
            } else {
              out.old = changed.oldCoords[1]
              out.coord = changed.newCoords[1]
            }
            return out
          }

          const maxChange = findMaxChange(changed)
          const finalCoords = setSimilarCoordinates(maxChange.old, maxChange.coord, oldCoordinates)
          console.log(finalCoords)
          console.log(modified.getGeometry().getCoordinates())
          modified.getGeometry().setCoordinates([finalCoords])

          // const feature = e.features.getArray()[0];
          // feature.un('change', modifySiblingCorners);
          this.currentVertex = null
        });
        this.map.addInteraction(this.modifyInteraction)

        // DRAW INTERACTION
        this.drawInteraction = new Draw({
          source: boxLayerSource,
          type: 'Circle',
          geometryFunction: createBox()
        })
      this.drawInteraction.on('drawend', (e,a,b,c,d) => {
        this.showLabelInput = true
        this.currentFeature = e.feature
      })

      function setSimilarCoordinates (oldVal, newVal, coordinates) {
          let out = []
          coordinates.forEach(coord => {
            let changed = coord
            if (changed[0] === oldVal) {
              changed[0] = newVal
            } else if (changed[1] === oldVal) {
              changed[1] = newVal
            }
            out.push(changed)
          })
        return out
      }

        this.map.addInteraction(this.drawInteraction)

        // this.transformInteraction = new ol.interaction.Transform( {
        //   enableRotatedTransform: false,
        //   hitTolerance: 2,
        //   stretch: true,
        //   translateFeature: false,
        //   scale: false,
        //   rotate: false,
        //   keepAspectRatio: false,
        //   translate: false,
        // });
        // this.transformInteraction.setStyle ('scaleh1',
        //         new Style({
        //           text: new Text ({
        //             text:'\uf07d',
        //             font:"bold 20px Fontawesome",
        //             fill: new Fill({ color:[255,255,255,0.8] }),
        //             stroke: new Stroke({ width:2, color:'red' })
        //           })
        //         }));
        // this.transformInteraction.style.scaleh3 = this.transformInteraction.style.scaleh1;
        // this.transformInteraction.setStyle('scalev',
        //         new Style({
        //           text: new Text ({
        //             text:'\uf07e',
        //             font:"bold 20px Fontawesome",
        //             fill: new Fill({ color:[255,255,255,0.8] }),
        //             stroke: new Stroke({ width:2, color:'red' })
        //           })
        //         }));
        // this.transformInteraction.style.scalev2 = this.transformInteraction.style.scalev;


        // Style handles
        // setHandleStyle();

        // this.selectInteraction = new Select()
        // this.map.addInteraction(this.selectInteraction);

        // SNAP INTERACTION
        this.map.addInteraction(new Snap({source: boxLayerSource}))

        // this.selectInteraction.on('select', (e) => {
        //   const feature = e.selected[0]
        //   console.log(feature)
        //   this.transformInteraction.select(feature, true)
        //   this.map.addInteraction(this.transformInteraction)
        // })
    }
  },

  mounted () {
    this.initMap()
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.map {
  width: 90vw;
  height: 120vh;

}
</style>
