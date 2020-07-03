<template>
  <div>
    <div id="map" class="map"></div>
<!--    <p>Current interaction: {{ this.currentInteraction }}</p>-->
<!--    <button @click="toggleInteraction">Toggle interaction</button>-->
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

// drawing
import {Draw, Select, Snap, Modify, defaults as defaultInteractions} from 'ol/interaction';
import {createBox} from 'ol/interaction/Draw';
import {Vector as VectorLayer} from 'ol/layer';
import {Vector as VectorSource} from 'ol/source';
import {Circle as CircleStyle, Fill, Stroke, Style} from 'ol/style';
import ol_interaction_Transform from 'ol-ext/interaction/Transform';
import throttle from 'lodash.throttle'

export default {
  name: 'ImageAnnotator',

  data () {
    return {
      map: null,
      currentInteraction: 'draw',
    }
  },

  methods: {
    // toggleInteraction () {
    //   if (this.currentInteraction === 'draw') {
    //     this.map.removeInteraction(this.drawInteraction)
    //     this.map.addInteraction(this.selectInteraction)
    //     this.currentInteraction = 'select'
    //   } else if (this.currentInteraction === 'select') {
    //     this.map.removeInteraction(this.selectInteraction)
    //     this.map.addInteraction(this.drawInteraction)
    //     this.currentInteraction = 'draw'
    //   }
    // },

    initMap() {
      // Map views always need a projection.  Here we just want to map image
      // coordinates directly to map coordinates, so we create a projection that uses
      // the image extent in pixels.
      const extent = [0, 0, 1024, 968];
      const projection = new Projection({
        code: 'xkcd-image',
        units: 'pixels',
        extent: extent
      });

      const boxLayerSource = new VectorSource();
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
          })
        })
      });

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
              url: 'https://imgs.xkcd.com/comics/online_communities.png',
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
      // this.modifyInteraction = new Modify({source: boxLayerSource, insertVertexCondition: () => false});
      // this.modifyInteraction.on('modifystart', (e) => {
      //   console.log('modify start')
      //   const feature = e.features.getArray()[0]
      //   const initCoordinates = feature.getGeometry().getCoordinates()[0]
      //
      //   feature.on('change', (e) => {
      //     const newCoords = e.target.getGeometry().getCoordinates()[0]
      //
      //     initCoordinates.forEach((point, idx) => {
      //       if (!this.currentVertex || this.currentVertex === -1) {
      //         this.currentVertex = newCoords.find((coordArray, idx) => {
      //           return coordArray[0] !== point[0] || coordArray[1] !== point[1]
      //         })
      //       }
      //     })
      //   });

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

        // this.modifyInteraction.on('modifyend', (e) => {
        //   console.log('modify end')
        //   // const feature = e.features.getArray()[0];
        //   // feature.un('change', modifySiblingCorners);
        //   this.currentVertex = null
        // });
        // this.map.addInteraction(this.modifyInteraction)

        // DRAW INTERACTION
        this.drawInteraction = new Draw({
          source: boxLayerSource,
          type: 'Circle',
          geometryFunction: createBox()
        })
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
  width: 1024px;
  height: 968px;

}
</style>
