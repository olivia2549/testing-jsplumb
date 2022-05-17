<template>
  <div id="app">
    <div class="container-fluid">
      <div class="row">
        <div class="col-md-3">
          <div id="toolbox" class="justify-content-center"></div>
        </div>
        <div class="col-md-9">
          <div class="map-container" id="map-container" style="position: relative;"></div>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
@import "https://code.jquery.com/ui/1.12.1/themes/base/jquery-ui.css";
@import "https://stackpath.bootstrapcdn.com/bootstrap/4.5.0/css/bootstrap.min.css";

/******  SETUP  ******/
.window { z-index: 20; }
.jtk-connector { z-index: 4; }
.jtk-endpoint { z-index: 5; }
.jtk-overlay { z-index: 6; }

/******  CUSTOM STYLES  ******/
#map-container, #toolbox {
  margin: 20px 0;
  border: 2px solid lightgray;
  height: 95vh;
}
#toolbox {
  display: flex;
  padding: 10px;
  flex-flow: row wrap;
  align-content: flex-start;
}
.context-menu {
  z-index: 1000;
  position: absolute;
  border: 1px solid black;
  padding: 2px;
}

</style>

<script>
import $ from 'jquery';
import { jsPlumb } from 'jsplumb';
import { loadScript } from "vue-plugin-load-script";

require('./assets/css/styles.css');
loadScript('https://code.jquery.com/jquery-3.6.0.min.js');

export default {
  name: 'MainApp',
  data() {
    return {
      data: {
        nodes: [
          {
            name: 'friction in the body',
          },
          {
            name: 'heat inside the body',
          },
          {
            name: 'body temperature',
          },
          {
            name: 'skeletal muscle contractions',
          },
        ],
      },
    };
  },
  methods: {
    createFlow(flowData) {
      const instance = jsPlumb.getInstance({
        Connector: 'Straight',
        Endpoint: 'Dot',
        DragOptions: { cursor: 'pointer', zIndex: 5000 },
        PaintStyle: { stroke: 'darkgray', strokeWidth: 2 },
        EndpointStyle: { radius: 9, fill: 'darkgray', stroke: '#000' },
        HoverPaintStyle: { stroke: 'blue', strokeWidth: 6 },
        ConnectionHoverStyle: { fill: 'blue', stroke: '#000' },
        EndpointHoverStyle: { fill: 'blue', stroke: '#000' },
        ConnectionOverlays: [
          ['Arrow', {
            location: 1,
            id: 'arrow',
            foldback: 0.3,
          }],
        ],
        Container: 'map-container',
      });

      // initialize nodes
      instance.batch(() => {
        for (const node of flowData.nodes) {
          $('.map-container').append(
            `<div id="${node.name}" class="node">${node.name}</div>`,
          );

          instance.addEndpoint(node.name, {
            anchor: ["Right", "BottomRight", "Bottom", "BottomLeft", "Left", "TopLeft", "Top", "TopRight"],
          }, {
            isSource: true,
            isTarget: true,
            dragAllowedWhenFull: false,
          });
        }

        for (const node of flowData.nodes) {
          instance.draggable(`${node.name}`, {"containment": true});
        }
      });

      // contextmenu means right clicked on connection or endpoint
      instance.bind("contextmenu", (component, event) => {
        // if right-clicking on connector
        if (component.hasClass("jtk-connector")) {
          event.preventDefault();
          // window.selectedConnection = component;
          $(`<div class="context-menu">
              <button class="delete-connection">Delete Connection</button>
            </div>`).appendTo("body").css({top: event.pageY, left: event.pageX});
        }
      });

      jsPlumb.fire('jsPlumbDemoLoaded', instance);
    },
  },
  mounted() {
    jsPlumb.ready(() => {
        this.createFlow(this.data);
    });
  },
};
</script>

