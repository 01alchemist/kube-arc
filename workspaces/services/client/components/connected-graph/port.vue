<template>
  <div
    :id="portId"
    class="port-container"
    @drop="onDrop"
    @dragover="onDragOver"
  >
    <!-- <div class="port fixed"></div> -->
    <div
      ref="port"
      class="port"
      draggable="true"
      :style="{
        left: anchorPoint.x + 'px',
        top: anchorPoint.y + 'px',
        background: color
      }"
      @dragstart="onDragStart"
      @drag="onDrag"
      @dragend="onDragEnd"
    ></div>
    <!-- :style="{ left: position.x + 'px', top: position.y + 'px' }" -->
    <div class="receptor"></div>
  </div>
</template>

<style lang="css">
.port-container {
  position: absolute;
  padding-bottom: 5px;
}
.port {
  width: 10px;
  height: 10px;
  background: #409eff;
  position: absolute;
  cursor: pointer;
  z-index: 2;
  transform: translate(5px, 5px);
}
.fixed {
  user-select: none;
  background: #f56c6c;
  z-index: 1;
}
.receptor {
  width:20px;
  height:20px;
  background: #d1d1d1;
  z-index: 0;
}
</style>

<script lang="ts">
import { Vue, Component, Prop } from 'vue-property-decorator'
import { Action } from 'vuex-class'
import { Point } from '../common/types/point'
import { PortEvent, ConnectionEvent } from './events'
import { PortConnection, findOffset } from './connection'
import GraphNode from './node.vue'

@Component({})
export default class NodePort extends Vue {
  @Prop({ default: () => -1 }) serial!: number
  @Prop({ default: () => null }) parentNode!: GraphNode
  @Prop({ default: () => false }) isDummy!: boolean
  // Store actions
  @Action('graph/getPort') getPort!: (portId: string) => any
  @Action('graph/addPort') addPort!: (port: NodePort) => string

  color = '#409eff'
  initialPosition = { x: 0, y: 0 }
  position = { ...this.initialPosition }
  anchorPoint = { ...this.initialPosition }
  isMoving = false
  isMultiPort = true
  portId: string = ''
  connection: PortConnection | null = null
  connectedPort: NodePort | null = null

  async mounted() {
    this.portId = await this.addPort(this)
    if (!this.isDummy) {
      console.log('Port ID: ', this.portId)
    }
  }

  onDragStart(event: DragEvent) {
    this.isMoving = true
    this.color = '#849eb1'
    const port: NodePort = this
    const portId = this.portId

    if (event.dataTransfer) {
      event.dataTransfer.dropEffect = 'move'
      event.dataTransfer.setData('text/plain', portId)
    }
    this.$emit(PortEvent.MOVE_START, { port })
  }

  onDrag(event: DragEvent) {
    this.$emit(PortEvent.MOVE, event)
  }

  onDragEnd(event: DragEvent) {
    if (this.isMoving) {
      this.reset()
    }
    this.isMoving = false
    this.$emit(PortEvent.MOVE_END, { port: this })
  }

  setPosition(position: Point) {
    this.position.x = position.x
    this.position.y = position.y
  }

  updatePosition(parentElement: HTMLElement) {
    const offset = findOffset(this.$el as HTMLElement, parentElement)
    this.position.x = offset.x + 10
    this.position.y = offset.y + 10
  }

  anchor(pos?: Point) {
    const position = pos || this.position
    this.color = '#f56c6c'
    this.isMoving = false
    this.anchorPoint.x = position.x
    this.anchorPoint.y = position.y
  }

  reset() {
    this.color = '#409eff'
    this.anchorPoint.x = this.initialPosition.x
    this.anchorPoint.y = this.initialPosition.y
  }

  onDragOver(event: any) {
    event.preventDefault()
    event.dataTransfer.dropEffect = 'move'
  }

  async onDrop(event: any) {
    if (this.isDummy) return
    event.preventDefault()
    event.stopPropagation()
    event.dataTransfer.dropEffect = 'move'
    const portId = event.dataTransfer.getData('text/plain')
    const droppedPort = await this.getPort(portId)
    this.dropPort(droppedPort)
  }

  dropPort(droppedPort) {
    if (this.connectedPort && this !== droppedPort) {
      if (!this.isMultiPort) {
        console.warn(`Port ${droppedPort.portId} not accepted`)
        this.$emit(ConnectionEvent.DISCONNECT, [droppedPort])
      } else {
        console.info(
          `Port multi-connect [${droppedPort.portId}, ${this.portId}]`
        )
        // If the port is CONNECT take the CONNECT port
        if (droppedPort.connectedPort) {
          const connectedPort = droppedPort.connectedPort
          droppedPort.connectedPort = null
          droppedPort.disconnect()
          droppedPort = connectedPort
        }
        this.$emit(ConnectionEvent.MULTI_CONNECT, {
          droppedPort,
          receivedPort: this
        })
      }
      return
    }

    // If the port is CONNECT take the CONNECT port
    if (droppedPort.connectedPort) {
      const connectedPort = droppedPort.connectedPort
      droppedPort.connectedPort = null
      droppedPort.disconnect()
      droppedPort = connectedPort
    }

    if (droppedPort === this) {
      this.$emit(ConnectionEvent.DISCONNECT, [this])
      return
    }

    if (droppedPort) {
      droppedPort.connect(this)
      this.connect(droppedPort)

      console.log(`🔌 Ports [${this.portId}, ${droppedPort.portId}] connected`)

      const connection: PortConnection =
        droppedPort.connection || this.connection

      if (connection) {
        connection.ports = [this, droppedPort]
      }

      this.connection = connection
      droppedPort.connection = connection
      this.$emit(ConnectionEvent.CONNECT, [this, droppedPort])
    }
  }

  connect(port: NodePort) {
    if (this.connection === null && port === this) return
    this.color = '#f56c6c'
    this.isMoving = false
    this.connectedPort = port
  }

  disconnect(ports?: NodePort[]) {
    if (!this.isDummy) {
      console.log(`Port ${this.portId} disconnected`)
    }

    this.isMoving = false
    this.connection = null
    const connectedPort = this.connectedPort
    this.connectedPort = null
    if (connectedPort) {
      connectedPort.disconnect()
    }
    this.reset()
    this.$emit(ConnectionEvent.DISCONNECTED, [this])
  }
}
</script>
