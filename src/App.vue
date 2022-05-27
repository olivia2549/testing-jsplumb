<template>
  <div id="app">
    <div class="container-fluid">
      <div class="row">
        <div class="col-md-3">
          <div id="toolbox" style="position: relative;">
            <div class="row justify-content-center">
                <div
                  v-for="node in data.nodesInBank"
                  :key="node.name+'_toolbox'"
                  :id="node.name+'_toolbox'"
                  ref="nodes"
                  class="node"
                  draggable="true"
                  @dragstart="onStartDrag($event, node)"
                >
                  {{ node.name }}
                </div>
            </div>
          </div>
        </div>
        <div class="col-md-9">
          <div 
            class="map-container" 
            id="map-container" 
            style="position: relative;"
            ref="map"
            droppable="true"
            @drop="onDrop($event)"
            @dragenter.prevent
            @dragover.prevent
            @drop.prevent
          >
              <div
                v-for="node in data.nodesOnMap"
                @contextmenu="event => openContextMenuForNode(node, event)"
                :key="node.name"
                :id="node.name"
                class="node"
                :style="{
                  left: node.x + 'px',
                  top: node.y + 'px',
                }"
              >
                {{ node.name }}
              </div>
          </div>
        </div>
      </div>
    </div>
    <!--- Context Menu Node --->
    <div
      v-if="contextMenuNode"
      class="context-menu"
      @mousedown="event => event.stopPropagation()"
      :style="{
        top: contextMenuNode.y + 'px',
        left: contextMenuNode.x + 'px'
      }"
    >
      <button @click="deleteNode(contextMenuNode.node)">delete</button>
    </div>
    <!--- Context Menu Connection --->
    <div
      v-if="contextMenuConnection"
      class="context-menu"
      @mousedown="event => event.stopPropagation()"
      :style="{
        top: contextMenuConnection.y + 'px',
        left: contextMenuConnection.x + 'px'
      }"
    >
      <button @click="deleteConnection(contextMenuConnection.connection)">delete</button>
    </div>
  </div>
</template>

<script>
import { jsPlumb } from 'jsplumb';

require('./assets/css/styles.css');

export default {
  name: 'MainApp',
  data() {
    return {
      contextMenuNode: null,
      contextMenuConnection: null,
      data: {
        nodesInBank: [
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
        nodesOnMap: [],
      },
    };
  },
  created() {
    window.addEventListener('mousedown', this.handleWindowClick);
  },
  destroyed() {
    window.removeEventListener('mousedown', this.handleWindowClick);
  },
  methods: {
    onStartDrag(event, node) {
      event.dataTransfer.dropEffect = 'move';
      event.dataTransfer.effectAllowed = 'move';
      event.dataTransfer.setData('itemID', node.name);
      const name = node.name;
      const idx = this.data.nodesInBank.findIndex(
        node => node.name === name
      );
      // find offset of mouse in relation to node
      this.$nextTick(() => {
        node.offsetX = event.clientX - this.$refs.nodes[idx].getBoundingClientRect().left;
        node.offsetY = event.clientY - this.$refs.nodes[idx].getBoundingClientRect().top;
      });
    },
    onDrop(event) {
      // find dropped node
      const name = event.dataTransfer.getData('itemID');
      const idx = this.data.nodesInBank.findIndex(
        node => node.name === name
      );
      const node = this.data.nodesInBank[idx];

      // create new node
      const newNode = {
        name,
        x: event.clientX - this.$refs.map.getBoundingClientRect().left - node.offsetX,
        y: event.clientY - this.$refs.map.getBoundingClientRect().top - node.offsetY,
      };

      // ensure new node positions inside the box
      if (newNode.x < 0) newNode.x = 0;
      if (newNode.y < 0) newNode.y = 0;

      // add new node to map and remove from bank
      this.data.nodesOnMap.push(newNode);
      this.data.nodesInBank.splice(idx, 1);

      // give the node jsplumb features
      this.$nextTick(() => {
        this.jsPlumbInstance.draggable(name, {containment: true});
        this.jsPlumbInstance.addEndpoint(name, {
          anchor: ["Right", "BottomRight", "Bottom", "BottomLeft", "Left", "TopLeft", "Top", "TopRight"],
          }, {
            isSource: true,
            isTarget: true,
            dragAllowedWhenFull: false,
        });
      });
    },
    deleteNode(node) {
      this.data.nodesOnMap = this.data.nodesOnMap.filter(n => n !== node);
      this.jsPlumbInstance.remove(node.name);
      // hide context menu
      this.contextMenuNode = null;
    },
    deleteConnection(connection) {
      this.jsPlumbInstance.deleteConnection(connection);
      // hide context menu
      this.contextMenuConnection = null;
    },
    handleConnection(info) {
      this.jsPlumbInstance.addEndpoint(info.sourceId, {
        anchor: ["Right", "BottomRight", "Bottom", "BottomLeft", "Left", "TopLeft", "Top", "TopRight"],
        }, {
          isSource: true,
          isTarget: true,
          dragAllowedWhenFull: false,
      });
      this.jsPlumbInstance.addEndpoint(info.targetId, {
        anchor: ["Right", "BottomRight", "Bottom", "BottomLeft", "Left", "TopLeft", "Top", "TopRight"],
        }, {
        isSource: true,
        isTarget: true,
        dragAllowedWhenFull: false,
      });
    },
    openContextMenuForNode(node, event) {
      event.preventDefault();
      this.contextMenuNode = {
        node,
        x: event.clientX,
        y: event.clientY
      }
    },
    openContextMenuForConnection(connection, event) {
      event.preventDefault();
      this.contextMenuConnection = {
        connection,
        x: event.clientX,
        y: event.clientY
      }
    },
    handleWindowClick() {
      this.contextMenuNode = null;
    },
    createFlow() {
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

      this.jsPlumbInstance = instance;

      // event right click on connector
      instance.bind("contextmenu", (component, event) => {
        if (component.hasClass("jtk-connector")) {
          event.preventDefault();
          this.openContextMenuForConnection(component, event);
        }
      });
      // add new endpoints when connection is made
      instance.bind("connection", (info) => {
        this.handleConnection(info);
      });

      jsPlumb.fire('jsPlumbDemoLoaded', instance);
    },
  },
  mounted() {
    jsPlumb.ready(() => {
        this.createFlow();
    });
  },
};
</script>

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
  padding: 10px;
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
