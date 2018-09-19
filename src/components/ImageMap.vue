<template>
  <div>
    <img
        class="d-none"
        :src="value.source"
        ref="image"
    >
    <el-button
        type="primary"
        @click="openDialog">Edit</el-button>
    <el-dialog
        title="ImageMap"
        :visible.sync="centerDialogVisible"
        width="1000px"
        :modal-append-to-body="false"
        center>
      <div class="d-flex justify-content-center align-items-center p-4 transparent-bg flex-column">
        <div :ref="'zonearea-wrap-'+value.uuid">
          <div
              class="zonearea"
              :style="{ backgroundImage: 'url(' + value.source + ')',
                      width:bgWidth+'px',height:bgHeight+'px' }"
              @mousedown.left.stop="createZone($event)"
              @mousemove="resizeZoneMove($event)"
              @mouseup="resizeZoneStop($event)"
          >
            <div
                v-for="(zone, index) in zones"
                class="zone"
                :key="index"
                :zone-id="index"
                :style="{
                width: zone.w + 'px',
                height: zone.h + 'px',
                top: (zone.y + bgTopBase) + 'px',
                left: (zone.x + bgLeftBase) + 'px',
                'z-index' : (zone.i || (index + 1))
              }"
                @mousedown.left.stop="moveItemRecord($event,index)"
                @mousemove="moveItem($event,index)"
                @mouseup="moveStop($event)"
                @contextmenu.prevent.stop="openZoneModify(index)"
            >
              <div
                  @mousedown.left.stop="resizeZoneStart($event, index, 'topleft')"
                  class="topleft"/>
              <div
                  @mousedown.left.stop="resizeZoneStart($event, index, 'top')"
                  class="top"/>
              <div
                  @mousedown.left.stop="resizeZoneStart($event, index, 'topright')"
                  class="topright"/>
              <div
                  @mousedown.left.stop="resizeZoneStart($event, index, 'right')"
                  class="right"/>
              <div
                  @mousedown.left.stop="resizeZoneStart($event, index, 'bottomright')"
                  class="bottomright"/>
              <div
                  @mousedown.left.stop="resizeZoneStart($event, index, 'bottom')"
                  class="bottom"/>
              <div
                  @mousedown.left.stop="resizeZoneStart($event, index, 'bottomleft')"
                  class="bottomleft"/>
              <div
                  @mousedown.left.stop="resizeZoneStart($event, index, 'left')"
                  class="left"/>
            </div>
          </div>

        </div>
        <div class="p-3"><h5>Left Click to drage or resize, Right Click to edit link</h5></div>
        <el-dialog
            width="30%"
            title="Edit ImageMap"
            :visible.sync="innerVisible"
            :modal = "false"
            center
        >
          <el-form
              @submit.native.prevent
              :model="form"
              label-width="50px"
          >
            <el-form-item label="Link">
              <el-input v-model="form.link"/>
            </el-form-item>
          </el-form>
          <span
              slot="footer"
              class="dialog-footer"
          >
            <el-button @click="handleZoneCancel()">Cancel</el-button>
            <el-button
                type="danger"
                @click="handleZoneRemove($event)"
                @keyup.enter.native="handleZoneRemove($event)"
            >
              Delete
            </el-button>
            <el-button
                type="primary"
                @click="handleZoneSubmit($event)"
                @keyup.enter.native="handleZoneSubmit($event)"
                native-type="submit">
              Confirm
            </el-button>
          </span>
        </el-dialog>
      </div>
      <span
          slot="footer"
          class="dialog-footer">
        <div>
          <el-button @click="handleMapCancel()">Cancel</el-button>
          <el-button
              type="primary"
              @click="handleMapSubmit()">Confirm</el-button>
        </div>
      </span>
    </el-dialog>
  </div>
</template>

<script>

  export default {
    name: 'ImageMap',
    props: {
      value: {
        type: Object,
        required: true,
      },
    },
    data() {
      return {
        centerDialogVisible: false,
        overRange: false,
        resizeZoneId: null,
        resizeZoneEl: null,
        resizeZoneType: null,
        resizeZoneLastXY: {
          x: 0,
          y: 0,
        },
        zones: this.value.zones || [],
        initZones: [],
        zoneMinSize: 4,
        dragging: false,
        beforeMoving: false,
        moving: false,
        tempIndex: null,
        moveXy: {
          x: 0,
          y: 0,
        },
        zoneXy: {
          x: 0,
          y: 0,
        },
        innerVisible: false,
        form: {
          link: null,
        },
        modifiedZoneIndex: null,
        bgWidth: 750,
        bgHeight: 240,
        bgTopBase: null,
        bgLeftBase: null,
      };
    },
    mounted() {
      if (this.zones) {
        const initZones = JSON.parse(JSON.stringify(this.zones));

        this.initZones = initZones;
      }
      this.$refs.image.onload = () => {
        this.bgWidth = this.$refs.image.naturalWidth;
        this.bgHeight = this.$refs.image.naturalHeight;
      };
    },
    methods: {
      openDialog() {
        this.centerDialogVisible = true;
        this.$nextTick(() => {
          const $zonearea = this.$refs[`zonearea-wrap-${this.value.uuid}`].querySelector('.zonearea');
          const dialog = this.$refs[`zonearea-wrap-${this.value.uuid}`].parentElement.parentElement.parentElement;
          this.bgLeftBase = $zonearea.getBoundingClientRect().x - dialog.getBoundingClientRect().x;
          this.bgTopBase =$zonearea.getBoundingClientRect().y - dialog.getBoundingClientRect().y;
        });
      },
      createZone(evt) {
        if (!this.beforeMoving) {
          if (evt.srcElement.classList.value.split(' ').indexOf('zonearea') === -1) {
            return;
          }
          const [x, y] = [evt.x, evt.y];
          const $zonearea = evt.srcElement.parentElement;
          const areaX = $zonearea.getBoundingClientRect().x;
          const areaY = $zonearea.getBoundingClientRect().y;
          const id = this.addZone({
            x: (x - areaX),
            y: (y - areaY),
          });
          this.$nextTick(() => {
            this.resizeZoneStart(evt, id, 'bottomright');
          });
        }
      },
      addZone(zone) {
        let newZone = Object.assign({
          w: this.zoneMinSize,
          h: this.zoneMinSize,
          x: 0,
          y: 0,
          i: 0,
          data: {},
        }, zone);
        newZone = this.zoneRangeFilter(newZone);
        const index = this.zones.push(newZone);
        return index - 1;
      },
      zoneRangeFilter(zone) {
        const newZone = zone;
        const $zonearea = this.$refs[`zonearea-wrap-${this.value.uuid}`].querySelector('.zonearea');
        const [clientWidth, clientHeight] = [$zonearea.clientWidth, $zonearea.clientHeight];
        if (zone.x < 0) {
          newZone.x = 0;
        }
        if (zone.x + zone.w > clientWidth) {
          if (zone.w < clientWidth) {
            newZone.x = clientWidth - zone.w;
          } else if (zone.x < clientWidth) {
            newZone.w = clientWidth - zone.x;
          } else {
            newZone.x = 0;
            newZone.w = clientWidth;
          }
        }

        if (zone.y < 0) {
          newZone.y = 0;
        }

        if (zone.y + zone.h > clientHeight) {
          if (zone.h < clientHeight) {
            newZone.y = clientHeight - zone.h;
          } else if (zone.y < clientHeight) {
            newZone.h = clientHeight - zone.y;
          } else {
            newZone.y = 0;
            newZone.h = clientHeight;
          }
        }

        if (zone.w < this.zoneMinSize) {
          newZone.w = this.zoneMinSize;
        }

        if (zone.h < this.zoneMinSize) {
          newZone.h = this.zoneMinSize;
        }
        return newZone;
      },
      resizeZoneStart(evt, id, type) {
        this.dragging = true;
        const $zone = this.$refs[`zonearea-wrap-${this.value.uuid}`].querySelector('.zonearea').querySelector(`[zone-id="${id}"]`);
        this.resizeZoneEl = $zone;
        this.resizeZoneId = id;
        this.resizeZoneType = type;
        this.resizeZoneLastXY.x = evt.x;
        this.resizeZoneLastXY.y = evt.y;
      },
      resizeZoneMove(evt) {
        if (this.dragging) {
          const zone = this.zones[this.resizeZoneId];
          const $zonearea = this.$refs[`zonearea-wrap-${this.value.uuid}`].querySelector('.zonearea');
          const $zone = this.$refs[`zonearea-wrap-${this.value.uuid}`].querySelector('.zonearea').querySelector(`[zone-id="${this.resizeZoneId}"]`);
          let x = +zone.x;
          let y = +zone.y;
          let w = +zone.w;
          let h = +zone.h;
          const evtx = evt.x;
          const evty = evt.y;
          let ox;
          let oy;
          let ow;
          let oh;
          this.resizeClean();
          this.resizeZoneEl.classList.add(`resize-${this.resizeZoneType}`);
          this.resizeZoneEl.classList.add('resize');
          $zonearea.classList.add(`resize-${this.resizeZoneType}`);

          if (/right/.test(this.resizeZoneType)) {
            ow = w;
            w += (evtx - this.resizeZoneLastXY.x);

            if (w < this.zoneMinSize) {
              this.resizeClean();

              this.resizeZoneType = this.resizeZoneType.replace('right', 'left');
              this.resizeZoneEl.classList.add(`resize-${this.resizeZoneType}`);
              this.resizeZoneEl.classList.add('resize');
              $zonearea.classList.add(`resize-${this.resizeZoneType}`);

              if (w > -this.zoneMinSize) {
                zone.w = this.zoneMinSize;
                zone.x -= zone.w;
                this.resizeZoneLastXY.x = $zone.getBoundingClientRect().x - zone.w;
              } else {
                zone.w = Math.abs(w);
                zone.x -= zone.w;
                this.resizeZoneLastXY.x = $zone.getBoundingClientRect().x - zone.w;
              }

              return;
            }

            if ((w + x) >= $zonearea.clientWidth) {
              w = $zonearea.clientWidth - x;
              this.overRange = true;
            } else {
              this.overRange = false;
            }
          }

          if (/left/.test(this.resizeZoneType)) {
            ox = x;
            ow = w;
            x += (evtx - this.resizeZoneLastXY.x);
            w -= (evtx - this.resizeZoneLastXY.x);

            if (w < this.zoneMinSize) {
              this.resizeClean();

              this.resizeZoneType = this.resizeZoneType.replace('left', 'right');
              this.resizeZoneEl.classList.add(`resize-${this.resizeZoneType}`);
              this.resizeZoneEl.classList.add('resize');
              $zonearea.classList.add(`resize-${this.resizeZoneType}`);

              if (w > -this.zoneMinSize) {
                zone.w = this.zoneMinSize;
                zone.x += ow;
                this.resizeZoneLastXY.x = $zone.getBoundingClientRect().x + ow;
              } else {
                zone.w = Math.abs(w + ow);
                zone.x += ow;
                this.resizeZoneLastXY.x = $zone.getBoundingClientRect().x + ow;
              }

              return;
            }

            if (x <= 0) {
              x = 0;
              w = ow - (x - ox);
              this.overRange = true;
            } else {
              this.overRange = false;
            }
          }

          if (/bottom/.test(this.resizeZoneType)) {
            oh = h;
            h += (evty - this.resizeZoneLastXY.y);

            if (h < this.zoneMinSize) {
              this.resizeClean();

              this.resizeZoneType = this.resizeZoneType.replace('bottom', 'top');
              this.resizeZoneEl.classList.add(`resize-${this.resizeZoneType}`);
              this.resizeZoneEl.classList.add('resize');
              $zonearea.classList.add(`resize-${this.resizeZoneType}`);

              if (h > -this.zoneMinSize) {
                zone.h = this.zoneMinSize;
                zone.y -= zone.h;
                this.resizeZoneLastXY.y = $zone.getBoundingClientRect().y - zone.h;
              } else {
                zone.h = Math.abs(h);
                zone.y -= zone.h;
                this.resizeZoneLastXY.y = $zone.getBoundingClientRect().y - zone.h;
              }

              return;
            }

            if ((h + y) >= $zonearea.clientHeight) {
              h = $zonearea.clientHeight - y;
              this.overRange = true;
            } else {
              this.overRange = false;
            }
          }

          if (/top/.test(this.resizeZoneType)) {
            oy = y;
            oh = h;
            y += (evty - this.resizeZoneLastXY.y);
            h -= (evty - this.resizeZoneLastXY.y);

            if (h < this.zoneMinSize) {
              this.resizeClean();
              this.resizeZoneType = this.resizeZoneType.replace('top', 'bottom');
              this.resizeZoneEl.classList.add(`resize-${this.resizeZoneType}`);
              this.resizeZoneEl.classList.add('resize');
              $zonearea.classList.add(`resize-${this.resizeZoneType}`);

              if (h > -this.zoneMinSize) {
                zone.h = this.zoneMinSize;
                zone.y += oh;
                this.resizeZoneLastXY.y = $zone.getBoundingClientRect().y + oh;
              } else {
                zone.h = Math.abs(h + oh);
                zone.y += oh;
                this.resizeZoneLastXY.y = $zone.getBoundingClientRect().y + oh;
              }

              return;
            }

            if (y <= 0) {
              y = 0;
              h = oh - (y - oy);
              this.overRange = true;
            } else {
              this.overRange = false;
            }
          }

          this.resizeZoneLastXY.x = evtx;
          this.resizeZoneLastXY.y = evty;

          zone.x = x;
          zone.y = y;
          zone.h = h;
          zone.w = w;
        }
      },
      resizeClean() {
        const $zonearea = this.$refs[`zonearea-wrap-${this.value.uuid}`].querySelector('.zonearea');
        this.resizeZoneEl.classList.remove('resize-top');
        this.resizeZoneEl.classList.remove('resize-bottom');
        this.resizeZoneEl.classList.remove('resize-left');
        this.resizeZoneEl.classList.remove('resize-right');
        this.resizeZoneEl.classList.remove('resize-topleft');
        this.resizeZoneEl.classList.remove('resize-topright');
        this.resizeZoneEl.classList.remove('resize-bottomleft');
        this.resizeZoneEl.classList.remove('resize-bottomright');
        this.resizeZoneEl.classList.remove('resize');

        $zonearea.classList.remove('resize-top');
        $zonearea.classList.remove('resize-bottom');
        $zonearea.classList.remove('resize-left');
        $zonearea.classList.remove('resize-right');
        $zonearea.classList.remove('resize-topleft');
        $zonearea.classList.remove('resize-topright');
        $zonearea.classList.remove('resize-bottomleft');
        $zonearea.classList.remove('resize-bottomright');
      },
      resizeZoneStop(evt) {
        if (this.dragging) {
          evt.stopPropagation();
          this.resizeClean();
          this.overRange = false;
          this.dragging = false;
        }
      },
      moveItemRecord(evt, index) {
        this.beforeMoving = true;
        this.moveXy = {
          x: evt.x,
          y: evt.y,
        };
        this.zoneXy = { x: this.zones[index].x, y: this.zones[index].y };
      },
      moveItem(evt, index) {
        if (this.beforeMoving && !this.dragging) {
          this.moving = true;
          const $zonearea = this.$refs[`zonearea-wrap-${this.value.uuid}`].querySelector('.zonearea');
          const [clientWidth, clientHeight] = [$zonearea.clientWidth, $zonearea.clientHeight];
          const moveX = this.zoneXy.x + (evt.x - this.moveXy.x);
          const moveY = this.zoneXy.y + (evt.y - this.moveXy.y);
          const maxX = clientWidth - this.zones[index].w;
          const maxY = clientHeight - this.zones[index].h;
          this.tempIndex = index;
          this.zones[index].i = 999;
          if (moveX > 0 && moveX < maxX) {
            this.zones[index].x = moveX;
          } else {
            this.moveStop();
          }
          if (moveY > 0 && moveY < maxY) {
            this.zones[index].y = moveY;
          } else {
            this.moveStop();
          }
        }
      },
      moveStop() {
        if (this.beforeMoving && this.moving && !this.dragging) {
          this.zones[this.tempIndex].i = this.tempIndex;
          this.tempIndex = null;
          this.beforeMoving = false;
          this.moveXy = { x: 0, y: 0 };
          this.zoneXy = { x: 0, y: 0 };
        }
      },
      openZoneModify(index) {
        this.modifiedZoneIndex = index;
        this.form.link = this.zones[this.modifiedZoneIndex].data.link;
        this.innerVisible = true;
      },
      handleZoneSubmit(evt) {
        evt.preventDefault();
        this.innerVisible = false;
        this.zones[this.modifiedZoneIndex].data.link = this.form.link;
        this.modifiedZoneIndex = null;
      },
      handleZoneRemove(evt) {
        evt.preventDefault();
        this.innerVisible = false;
        this.zones.splice(this.modifiedZoneIndex, 1);
        this.modifiedZoneIndex = null;
      },
      handleZoneCancel() {
        this.innerVisible = false;
        this.form.link = this.zones[this.modifiedZoneIndex].data.link;
        this.modifiedZoneIndex = null;
      },
      handleMapSubmit() {
        this.centerDialogVisible = false;
        this.$emit('input', this.zones);
      },
      handleMapCancel() {
        this.zones = JSON.parse(JSON.stringify(this.initZones));
        this.centerDialogVisible = false;
      },
    },
  };
</script>

<style lang="scss" scoped>
  $grid-bg: #ccc;
  .transparent-bg {
    background-size: 60px 60px;
    background-position: 0 0, 30px 30px;
    background-image: linear-gradient(45deg, $grid-bg 25%, transparent 25%,
        transparent 75%, $grid-bg 75%, $grid-bg),
    linear-gradient(45deg, $grid-bg 25%, transparent 25%,
            transparent 75%, $grid-bg 75%, $grid-bg);
  }

  .zonearea {
    display: block;
    cursor: crosshair;
    font-size: 14px;
    /*width: 750px;*/
    /*height:240px;*/
    border: 1px #EDF1F5 solid;
    box-sizing: content-box;

    &.disable-add-spot {
      cursor: default;
    }

    &.over-range {
      box-shadow: 0 0 6px rgba(214, 68, 49, 0.55) inset;
      border-color: #D64431;
    }
    .zone {
      box-sizing: border-box;
      border: 1px dashed #45505C;
      position: absolute;
      display: block;
      background-color: #45505C;
      opacity: 0.4;
      cursor: move;
      user-select: none;

      &.resize-top {
        border-top-style: solid !important;
        border-top-color: #D64431 !important;
      }

      &.resize-bottom {
        border-bottom-style: solid !important;
        border-bottom-color: #D64431 !important;
      }

      &.resize-left {
        border-left-style: solid !important;
        border-left-color: #AA3627 !important;
      }

      &.resize-right {
        border-right-style: solid !important;
        border-right-color: #AA3627 !important;
      }

      &.resize-topleft {
        border-top-style: solid !important;
        border-top-color: #AA3627 !important;
        border-left-style: solid !important;
        border-left-color: #AA3627 !important;
      }

      &.resize-bottomleft {
        border-bottom-style: solid !important;
        border-bottom-color: #AA3627 !important;
        border-left-style: solid !important;
        border-left-color: #AA3627 !important;
      }

      &.resize-topright {
        border-top-style: solid !important;
        border-top-color: #AA3627 !important;
        border-right-style: solid !important;
        border-right-color: #AA3627 !important;
      }

      &.resize-bottomright {
        border-bottom-style: solid !important;
        border-bottom-color: #AA3627 !important;
        border-right-style: solid !important;
        border-right-color: #AA3627 !important;
      }

      &:hover,
      &.resize {
        border: 1px dashed #D64431;
        background-color: #1e77ba;
        opacity: 0.3;
        border-width: 2px;
      }

      &.move-moving {
        display: none;
      }

      .top,
      .bottom {
        position: absolute;
        left: 0;
        width: 100%;
        height: 20px;
      }

      .top {
        top: -10px;
      }

      .bottom {
        bottom: -10px;
        z-index: 3;
      }

      .left,
      .right {
        position: absolute;
        top: 0;
        height: 100%;
        width: 20px;
      }

      .left {
        left: -10px;
      }

      .right {
        right: -10px;
        z-index: 3;
      }

      .topleft,
      .topright,
      .bottomleft,
      .bottomright {
        position: absolute;
        width: 20px;
        height: 20px;
        z-index: 4;
      }

      .topleft {
        top: -10px;
        left: -10px;
      }

      .topright {
        top: -10px;
        right: -10px;
      }

      .bottomleft {
        bottom: -10px;
        left: -10px;
      }

      .bottomright {
        bottom: -10px;
        right: -10px;
        z-index: 10;
      }
    }

    &.resize-top,
    & .zone .top {
      cursor: n-resize !important;
    }

    &.resize-bottom,
    & .zone .bottom {
      cursor: s-resize !important;
    }

    &.resize-left,
    & .zone .left {
      cursor: w-resize !important;
    }

    &.resize-right,
    & .zone .right {
      cursor: e-resize !important;
    }

    &.resize-topleft,
    & .zone .topleft {
      cursor: nw-resize !important;
    }

    &.resize-topright,
    & .zone .topright {
      cursor: ne-resize !important;
    }

    &.resize-bottomleft,
    & .zone .bottomleft {
      cursor: sw-resize !important;
    }

    &.resize-bottomright,
    & .zone .bottomright {
      cursor: se-resize !important;
    }
  }
</style>
