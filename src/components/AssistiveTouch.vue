<template>
  <div
    class="assistive-touch-container"
    ref="touchRef"
    :style="style"
    @mousedown="onMouseDown($event)"
    @touchstart="onTouchStart($event)"
  >
    <slot>
       <div class="assistive-touch-default-content"></div>
    </slot>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from 'vue-property-decorator';

const DEFAULT_TRANSITION_TIME = 0.3;

interface Position {
  top: number;
  left: number;
}

interface ScreenSize {
  width:number;
  height:number;
}

@Component
export default class AssistiveTouch extends Vue {
  @Prop() private transitionTime!: number;

  $refs!: {
    touchRef: HTMLDivElement;
  };

  lastMousePosition!: MouseEvent | Touch

  positionChanged = false;

  position: Position = {
    top: 0,
    left: 0,
  };

  prePos = {} as Position;

  get style(): Pick<CSSStyleDeclaration, 'top'|'left'> {
    const { left, top } = this.position;
    return {
      top: `${top}px`,
      left: `${left}px`,
    };
  }

  getScreenSize = ():ScreenSize => {
    const { clientWidth, clientHeight } = document.documentElement;
    const { innerWidth = 0, innerHeight = 0 } = window;
    return {
      width: Math.max(
        clientWidth,
        innerWidth,
      ),
      height: Math.max(
        clientHeight,
        innerHeight,
      ),
    };
  }

  onMouseDown(event: MouseEvent | Touch) {
    this.positionChanged = false;
    this.prePos.left = event.clientX;
    this.prePos.top = event.clientY;
    window.addEventListener('mousemove', this.onMouseMove);
    window.addEventListener('mouseup', this.onMouseUp);
    window.addEventListener('touchmove', this.onTouchMove);
    window.addEventListener('touchend', this.onTouchEnd);
    window.addEventListener('touchcancel', this.onTouchEnd);
  }

  onMouseMove = (event: MouseEvent | Touch) => {
    this.$refs.touchRef.style.transition = 'none';
    const { width: screenSizeWidth, height: screenSizeHeight } = this.getScreenSize();
    const { clientX: eventClientX, clientY: eventClientY } = event;
    const { left: positionLeft, top: positionTop } = this.position;
    const { left: prePosLeft, top: prePosTop } = this.prePos;
    const {
      clientWidth: touchRefClientWidth,
      clientHeight: touchRefClientHeight,
    } = this.$refs.touchRef;

    const diffPos: Position = {
      left:
        // eslint-disable-next-line no-nested-ternary
        eventClientX <= screenSizeWidth
          ? eventClientX <= 0
            ? positionLeft
            : prePosLeft - eventClientX
          : 0,
      top:
        // eslint-disable-next-line no-nested-ternary
        eventClientY <= screenSizeHeight
          ? eventClientY <= 0
            ? positionTop
            : prePosTop - eventClientY
          : 0,
    };

    this.positionChanged = true;
    let left = this.position.left - diffPos.left;
    if (left > screenSizeWidth - touchRefClientWidth) {
      left = screenSizeWidth - touchRefClientWidth;
    }
    let top = positionTop - diffPos.top;
    if (top > screenSizeHeight - touchRefClientHeight) {
      top = screenSizeHeight - touchRefClientHeight;
    }

    this.position.left = left < 0 ? 0 : left;
    this.position.top = top < 0 ? 0 : top;
    this.prePos.left = eventClientX;
    this.prePos.top = eventClientY;
  }

  onMouseUp = (event: MouseEvent | Touch) => {
    this.$refs.touchRef.style.transition = `all ${this.transitionTime || DEFAULT_TRANSITION_TIME}s ease`;
    this.lastMousePosition = event;
    if (this.positionChanged) {
      this.snapToSide(event);
    }
    this.removeListeners();
    this.positionChanged = false;
  }

  onTouchStart = (event: TouchEvent) => {
    event.preventDefault();
    this.onMouseDown(event.changedTouches[0]);
  }

  onTouchMove = (event: TouchEvent) => {
    event.preventDefault();
    this.onMouseMove(event.changedTouches[0]);
  }

  onTouchEnd = (event: TouchEvent) => {
    event.preventDefault();
    this.onMouseUp(event.changedTouches[0]);
  }

  removeListeners = () => {
    window.removeEventListener('mousemove', this.onMouseMove);
    window.removeEventListener('mouseup', this.onMouseUp);
    window.removeEventListener('touchmove', this.onTouchMove);
    window.removeEventListener('touchend', this.onTouchEnd);
    window.removeEventListener('touchcancel', this.onTouchEnd);
  }

  snapToSide = (event: MouseEvent | Touch) => {
    let { left, top } = this.position;
    const { clientX: eventClientX, clientY: eventClientY } = event;
    const { width: screenSizeWidth, height: screenSizeHeight } = this.getScreenSize();
    const {
      clientWidth: touchRefClientWidth,
      clientHeight: touchRefClientHeight,
    } = this.$refs.touchRef;

    // the leave point closer to the bottom
    if (eventClientY <= screenSizeHeight / 2) {
    // the leave point closer to the left
      if (eventClientX <= screenSizeWidth / 2) {
        // should move up or down, rather than left or right
        if (eventClientY <= eventClientX) {
          top = 0;
        } else {
          left = 0;
        }
      } else if (eventClientY <= screenSizeWidth - eventClientX) {
        top = 0;
      } else {
        left = screenSizeWidth - touchRefClientWidth;
      }
    } else if (eventClientX <= screenSizeWidth / 2) {
      if (screenSizeHeight - eventClientY <= eventClientX) {
        top = screenSizeHeight - touchRefClientHeight;
      } else {
        left = 0;
      }
    } else if (screenSizeHeight - eventClientY <= screenSizeWidth - eventClientX) {
      top = screenSizeHeight - touchRefClientHeight;
    } else {
      left = screenSizeWidth - touchRefClientWidth;
    }

    this.position.left = left;
    this.position.top = top;
  }
}
</script>

<style scoped>
.assistive-touch-container {
    position: fixed;
    width: 35px;
    height: 35px;
    background-color: rgba(0, 0, 0, 0.2);
    z-index: 10000;
    cursor: pointer;
    top: 0px;
    left: 0px;
    border-radius: 30%;
    padding: 6px;
    touch-action: none;
    cursor: grab;
    border: 1px solid rgba(0, 0, 0, 0.1);
}

.assistive-touch-default-content {
    border-radius: 50%;
    width: 100%;
    height: 100%;
    background-color: rgba(0, 0, 0, 0.3);
    pointer-events: none;
}
</style>
