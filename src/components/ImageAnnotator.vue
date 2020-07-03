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

export default {
  name: 'ImageAnnotator',

  data () {
    return {
      map: null,
      currentInteraction: 'draw',
      currentFeature: null,
      labelText: '',
      showLabelInput: false,
      currentFeatures: {},
      placeholder: null,
      isStyleApplied: false,
      initialStyle: {}
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
            'properties': {'score': 0.9962388277053833, 'label': 'Crossheading'}},
          {'type': 'Feature',
          'id': 'placeholder',
            'geometry': {'type': 'Polygon',
              'coordinates': [[[0, 0],
                [0, 0],
                [0, 0],
                [0, 0],
                [0, 0]]]},}]}

      const boxLayerSource = new VectorSource({
        features: (new GeoJSON()).readFeatures(testGeoJSON),
        format: new GeoJSON()
      })
      this.placeholder = boxLayerSource.getFeatureById('placeholder')
      this.initialStyle = new Style({
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
      const boxLayer = new VectorLayer({
        source: boxLayerSource,
        style: this.initialStyle
      });

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
      const applyFeatureUpdate = (e) => {
        const feature = e.target
        const featureId = feature.getId()
        const currentCoordinates = feature.getGeometry().getCoordinates()[0]
        let initCoords = JSON.parse(JSON.stringify(this.currentFeatures[featureId].initialCoords))

        // make original feature invisible for the duration of the modify interaction
        // do it only once
        if (!this.isStyleApplied) {
          const style = new Style({
            fill: new Fill({
              color: 'rgba(255, 255, 255, 0.2)'
            }),
            stroke: new Stroke({
              color: '#ffffff',
              width: 2
            }),
          })
          // always unregister the change event listener on feature before changing the feature in a way that would cause an infinite loop
          feature.un('change', applyFeatureUpdate)
          feature.setStyle(style)
          this.isStyleApplied = true
          // re-register event listener when done changing the feature
          feature.on('change', applyFeatureUpdate)
        }
        // use a placeholder geometry that gets the squarified-version of the changed coordinates as its extent
        const changed = getDifferentCoords(currentCoordinates, initCoords)
        if (changed) {
          feature.un('change', applyFeatureUpdate)
          const max = findMaxChange(changed)
          initCoords = setSimilarCoordinates(max.old, max.coord, initCoords)
          this.placeholder.getGeometry().setCoordinates([initCoords])
          feature.on('change', applyFeatureUpdate)
        }
      }

      const getDifferentCoords = (currentCoords, initCoords) => {
        let hasChanged = false
        currentCoords.forEach((coordpair, idx) => {
          if (coordpair[0] !== initCoords[idx][0] || coordpair[1] !== initCoords[idx][1]) {
            hasChanged = {}
            hasChanged.newCoords = coordpair
            hasChanged.oldCoords = initCoords[idx]
            hasChanged.idx = idx
          }
        })
        return hasChanged
      }

      const findMaxChange = (changed) => {
        let out = {}
        let firstDiff = Math.abs(changed.newCoords[0] - changed.oldCoords[0])
        let secondDiff = Math.abs(changed.newCoords[1] - changed.oldCoords[1])
        if (firstDiff > secondDiff) {
          out.old = changed.oldCoords[0]
          out.coord = changed.newCoords[0]
        } else if (firstDiff < secondDiff) {
          out.old = changed.oldCoords[1]
          out.coord = changed.newCoords[1]
        } else {
          out.old = changed.newCoords[1]
          out.coord = changed.newCoords[1]
        }
        return out
      }

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

      this.modifyInteraction.on('modifystart', (e) => {
        console.log('modify start')
        this.currentFeatures = e.features.getArray().reduce((acc, val) => {
          return {
            ...acc,
            [val.getId()]: {
              revision: val.getRevision(),
              initialCoords: JSON.parse(JSON.stringify(val.getGeometry().getCoordinates()[0]))
            }
          }
        }, {})

        // register change event listeners on all features except placeholder so that we can execute code when a feature is modified
        e.features.getArray().forEach(feature => {
          if(feature.getId() !== 'placeholder') feature.on('change', applyFeatureUpdate)
        })
      })

        this.modifyInteraction.on('modifyend', (e) => {
          console.log('modify end')

          // find the modified feature through its revision number
          const modified = e.features.getArray().reduce((acc, val) => {
            const oldFeatureRevision = this.currentFeatures[val.getId()].revision
            return val.getId() !== 'placeholder' && oldFeatureRevision !== val.getRevision() ? val : acc
          }, {})

          // squareify the original feature geometry
          const currentCoordinates = this.placeholder.getGeometry().getCoordinates()
          modified.getGeometry().setCoordinates(currentCoordinates)

          // reset styles and placeholder
          modified.setStyle(this.initialStyle)
          this.placeholder.getGeometry().setCoordinates([[[0, 0], [0, 0], [0, 0], [0, 0]]])
          this.isStyleApplied = false
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

        this.map.addInteraction(this.drawInteraction)

        //this.map.addInteraction(new Snap({source: boxLayerSource}))
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
