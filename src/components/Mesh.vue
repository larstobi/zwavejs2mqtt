<template>
  <v-container fluid>
    <v-card>
      <v-container grid-list-md>
        <v-layout row wrap>
          <v-flex xs2 md2>
            <v-text-field
              label="Nodes size"
              v-model.number="nodeSize"
              min="10"
              type="number"
            ></v-text-field>
          </v-flex>
          <v-flex xs2 md2>
            <v-text-field
              label="Font size"
              v-model.number="fontSize"
              min="10"
              type="number"
            ></v-text-field>
          </v-flex>
          <v-flex xs3 md2>
            <v-text-field
              label="Distance"
              v-model.number="force"
              min="100"
              type="number"
            ></v-text-field>
          </v-flex>
          <v-flex xs3 md2>
            <v-switch label="Show location" v-model="showLocation"></v-switch>
          </v-flex>
          <v-flex xs5 md6>
            <v-btn color="success" @click="downloadSVG">Download SVG</v-btn>
          </v-flex>
        </v-layout>
      </v-container>

      <d3-network
        id="mesh"
        ref="mesh"
        :net-nodes="activeNodes"
        :net-links="links"
        :options="options"
        :selection="selection"
        @node-click="nodeClick"
      />

      <div id="properties" draggable v-show="showProperties" class="details">
        <v-icon
          @click="showProperties = false"
          style="cursor:pointer;position:absolute;right:10px;top:10px"
          >clear</v-icon
        >
        <v-subheader>Node properties</v-subheader>
        <v-list
          v-if="selectedNode"
          dense
          style="min-width:300px;background:transparent"
        >
          <v-list-item>
            <v-list-item-content>ID</v-list-item-content>
            <v-list-item-content class="align-end">{{
              selectedNode.data.id
            }}</v-list-item-content>
          </v-list-item>
          <v-list-item>
            <v-list-item-content>Status</v-list-item-content>
            <v-list-item-content class="align-end">{{
              selectedNode.data.status
            }}</v-list-item-content>
          </v-list-item>
          <v-list-item>
            <v-list-item-content>Code</v-list-item-content>
            <v-list-item-content class="align-end">{{
              selectedNode.data.productLabel
            }}</v-list-item-content>
          </v-list-item>
          <v-list-item>
            <v-list-item-content>Product</v-list-item-content>
            <v-list-item-content class="align-end">{{
              selectedNode.data.productDescription
            }}</v-list-item-content>
          </v-list-item>
          <v-list-item>
            <v-list-item-content>Manufacturer</v-list-item-content>
            <v-list-item-content class="align-end">{{
              selectedNode.data.manufacturer
            }}</v-list-item-content>
          </v-list-item>
          <v-list-item>
            <v-list-item-content>Name</v-list-item-content>
            <v-list-item-content class="align-end">{{
              selectedNode.data.name
            }}</v-list-item-content>
          </v-list-item>
          <v-list-item>
            <v-list-item-content>Location</v-list-item-content>
            <v-list-item-content class="align-end">{{
              selectedNode.data.loc
            }}</v-list-item-content>
          </v-list-item>
        </v-list>
      </div>

      <v-speed-dial bottom fab right fixed v-model="fab">
        <template v-slot:activator>
          <v-btn color="blue darken-2" dark fab hover v-model="fab">
            <v-icon v-if="fab">close</v-icon>
            <v-icon v-else>add</v-icon>
          </v-btn>
        </template>
        <v-btn fab dark small color="green" @click="refresh">
          <v-icon>refresh</v-icon>
        </v-btn>
      </v-speed-dial>
    </v-card>
  </v-container>
</template>
<script>
import D3Network from 'vue-d3-network'

import { socketEvents, inboundEvents as socketActions } from '@/plugins/socket'

export default {
  name: 'Mesh',
  props: {
    socket: Object
  },
  components: {
    D3Network
  },
  watch: {
    showLocation () {
      for (const n of this.nodes) {
        n.name = this.nodeName(n)
      }
    }
  },
  computed: {
    activeNodes () {
      return this.nodes.filter(n => n.id !== 0 && n.status !== 'Removed')
    },
    options () {
      return {
        canvas: false,
        force: this.force,
        offset: {
          x: 0,
          y: 0
        },
        nodeSize: this.nodeSize,
        fontSize: this.fontSize,
        linkWidth: 1,
        nodeLabels: true,
        linkLabels: false,
        strLinks: true,
        resizeListener: true
      }
    },
    selection () {
      var s = {
        nodes: [],
        links: []
      }

      if (this.selectedNode) {
        s.nodes[this.selectedNode.id] = this.selectedNode
      }

      return s
    }
  },
  data () {
    return {
      nodeSize: 20,
      fontSize: 10,
      force: 2000,
      nodes: [],
      links: [],
      fab: false,
      selectedNode: null,
      showProperties: false,
      showLocation: false
    }
  },
  methods: {
    showSnackbar (text) {
      this.$emit('showSnackbar', text)
    },
    nodeClick (e, node) {
      this.selectedNode = this.selectedNode === node ? null : node
      this.showProperties = !!this.selectedNode
    },
    downloadSVG () {
      this.$refs.mesh.screenShot('myNetwork.svg', true, true)
    },
    convertNode (n) {
      return {
        id: n.id,
        _cssClass: this.nodeClass(n),
        name: this.nodeName(n),
        status: n.status,
        data: n
      }
    },
    nodeName (n) {
      if (n.data) n = n.data // works both with node object and mesh node object
      var name = n.name || n.product || 'node ' + n.id
      return name + (this.showLocation && n.loc ? ` (${n.loc})` : '')
    },
    nodeClass (n) {
      if (n.id === 1) {
        return 'controller'
      }
      return n.status.toLowerCase()
    },
    apiRequest (apiName, args) {
      if (this.socket.connected) {
        var data = {
          api: apiName,
          args: args
        }
        this.socket.emit(socketActions.zwave, data)
      } else {
        this.showSnackbar('Socket disconnected')
      }
    },
    refresh () {
      this.socket.emit(socketActions.zwave, {
        api: 'refreshNeighbors',
        args: []
      })
    },
    updateLinks () {
      this.links = []

      for (const source of this.nodes) {
        if (source.neighbors) {
          for (const target of source.neighbors) {
            this.links.push({
              sid: source.id,
              tid: target,
              _color: this.$vuetify.theme.dark ? 'white' : 'black'
            })
          }
        }
      }
    }
  },
  mounted () {
    var self = this

    this.socket.on(socketEvents.nodeRemoved, node => {
      self.$set(self.nodes, node.id, node)
    })

    this.socket.on(socketEvents.init, data => {
      var nodes = data.nodes
      for (var i = 0; i < nodes.length; i++) {
        self.nodes.push(self.convertNode(nodes[i]))
      }
    })

    this.socket.on(socketEvents.nodeRemoved, node => {
      self.$set(self.nodes, node.id, node)
      self.refresh()
    })

    this.socket.on(socketEvents.nodeUpdated, data => {
      var node = self.convertNode(data)

      // node added
      var refresh = !self.nodes[data.id] || self.nodes[data.id].failed

      // add missing nodes if new node added
      while (self.nodes.length < data.id) {
        self.nodes.push({
          id: self.nodes.length,
          failed: true,
          status: 'Removed'
        })
      }

      self.$set(self.nodes, data.id, node)

      // update links if new node has been added
      if (refresh) {
        self.refresh()
      }
    })

    this.socket.on(socketEvents.api, data => {
      if (data.success) {
        switch (data.api) {
          case 'refreshNeighbors':
            var neighbors = data.result
            for (var i = 0; i < neighbors.length; i++) {
              if (self.nodes[i]) {
                self.nodes[i].neighbors = neighbors[i]
              }
            }
            self.updateLinks()
            break
        }
      } else {
        self.showSnackbar(
          'Error while calling api ' + data.api + ': ' + data.message
        )
      }
    })

    this.socket.emit(socketActions.init, true)
    this.refresh()

    // make properties window draggable
    var propertiesDiv = document.getElementById('properties')
    var mesh = document.getElementById('mesh')
    var isDown = false
    var offset = [0, 0]

    // TODO: Update dimensions on screen resize
    var dimensions = [mesh.clientWidth, mesh.clientHeight]

    propertiesDiv.addEventListener(
      'mousedown',
      function (e) {
        isDown = true
        offset = [
          propertiesDiv.offsetLeft - e.clientX,
          propertiesDiv.offsetTop - e.clientY
        ]
      },
      true
    )

    document.addEventListener(
      'mouseup',
      function () {
        isDown = false
      },
      true
    )

    document.addEventListener(
      'mousemove',
      function (e) {
        event.preventDefault()
        if (isDown) {
          var l = e.clientX
          var r = e.clientY

          if (l > 0 && l < dimensions[0]) {
            propertiesDiv.style.left = l + offset[0] + 'px'
          }
          if (r > 0 && r < dimensions[1]) {
            propertiesDiv.style.top = r + offset[1] + 'px'
          }
        }
      },
      true
    )
  },
  beforeDestroy () {
    if (this.socket) {
      // unbind events
      for (const event in socketEvents) {
        this.socket.off(event)
      }
    }
  }
}
</script>
