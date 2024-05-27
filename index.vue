<script>
export default {
  name: 'FastMarquee',
  props: {
    className: {
      type: String,
      default: ''
    },
    autoFill: {
      type: Boolean,
      default: false
    },
    play: {
      type: Boolean,
      default: true
    },
    pauseOnHover: {
      type: Boolean,
      default: true
    },
    pauseOnClick: {
      type: Boolean,
      default: false
    },
    direction: {
      type: String,
      default: "left"
    },
    speed: {
      type: Number,
      default: 50
    },
    delay: {
      type: Number,
      default: 0
    },
    loop: {
      type: Number,
      default: 0
    },
    gradient: {
      type: Boolean,
      default: true
    },
    gradientColor: {
      type: String,
      default: "white"
    },
    gradientWidth: {
      type: Number,
      default: 100
    },
    onFinish: {
      type: Function,
      default: function () {}
    },
    onCycleComplete: {
      type: Function,
      default: function () {}
    },
  },
  data() {
    return {
      multiplier: 1,
      containerWidth: 0,
      marqueeWidth: 0
    }
  },
  mounted() {
    this.$watch(() => {
      return [this.autoFill, this.direction, this.$slots.default]
    }, () => {
      this.calculateWidth();
    }, {
      immediate: true
    });
    this.watchReSize();
  },
  destroyed() {
    this._resizeObserver && this._resizeObserver.disconnect();
  },
  computed: {
    duration() {
      if (this.autoFill) {
        return (this.marqueeWidth * this.multiplier) / this.speed;
      } else {
        return this.marqueeWidth < this.containerWidth
          ? this.containerWidth / this.speed
          : this.marqueeWidth / this.speed;
      }
    },
    containerStyle() {
      const {play, pauseOnHover, pauseOnClick, direction} = this;
      return {
        ["--pause-on-hover"]:
          !play || pauseOnHover ? "paused" : "running",
        ["--pause-on-click"]:
          !play || (pauseOnHover && !pauseOnClick) || pauseOnClick
            ? "paused"
            : "running",
        ["--width"]:
          direction === "up" || direction === "down" ? `100vh` : "100%",
        ["--transform"]:
          direction === "up"
            ? "rotate(-90deg)"
            : direction === "down"
            ? "rotate(90deg)"
            : "none",
      }
    },
    gradientStyle() {
      const {gradientColor, gradientWidth} = this;
      return {
        ["--gradient-color"]: gradientColor,
        ["--gradient-width"]:
          typeof gradientWidth === "number"
            ? `${gradientWidth}px`
            : gradientWidth,
      };
    },
    marqueeStyle() {
      const {play, duration, direction, loop, autoFill, delay} = this;
      return {
        ["--play"]: play ? "running" : "paused",
        ["--direction"]: direction === "left" ? "normal" : "reverse",
        ["--duration"]: `${duration}s`,
        ["--delay"]: `${delay}s`,
        ["--iteration-count"]: !!loop ? `${loop}` : "infinite",
        ["--min-width"]: autoFill ? `auto` : "100%",
      };
    },
    childStyle() {
      const {direction} = this;
      return {
        ["--transform"]:
          direction === "up"
            ? "rotate(90deg)"
            : direction === "down"
            ? "rotate(-90deg)"
            : "none",
      }
    }
  },
  methods: {
    watchReSize() {
      const {marqueeRef, containerRef} = this.$refs;
      if (marqueeRef && containerRef) {
        const resizeObserver = new ResizeObserver(() => this.calculateWidth());
        resizeObserver.observe(containerRef);
        resizeObserver.observe(marqueeRef);
        this._resizeObserver = resizeObserver;
      }
    },
    calculateWidth() {
      const {marqueeRef, containerRef} = this.$refs
      const {direction, autoFill} = this;
      if (marqueeRef && containerRef) {
        const containerRect = containerRef.getBoundingClientRect();
        const marqueeRect = marqueeRef.getBoundingClientRect();
        let containerWidth = containerRect.width;
        let marqueeWidth = marqueeRect.width;
        // Swap width and height if direction is up or down
        if (direction === "up" || direction === "down") {
          containerWidth = containerRect.height;
          marqueeWidth = marqueeRect.height;
        }
        if (autoFill && containerWidth && marqueeWidth) {
          this.multiplier = marqueeWidth < containerWidth
            ? Math.ceil(containerWidth / marqueeWidth)
            : 1;
        } else {
          this.multiplier = 1;
        }
        this.containerWidth = containerWidth;
        this.marqueeWidth = marqueeWidth;
      }
    },
    renderChild() {
      return <div style={this.childStyle} className="rfm-child">
        {this.$slots.default}
      </div>
    },
    renderMultiple(multiplier) {
      return [
        ...Array(
          Number.isFinite(multiplier) && multiplier >= 0 ? multiplier : 0
        ),
      ].map((_, i) => {
        return this.renderChild()
      });
    },
  },
  render(h) {
    const {
      containerStyle,
      className,
      gradientStyle,
      gradient,
      marqueeStyle,
      onCycleComplete,
      onFinish,
      childStyle
    } = this;
    return <div
      ref="containerRef"
      style={containerStyle}
      class={"rfm-marquee-container " + className}
    >
      {gradient && <div style={gradientStyle} class="rfm-overlay"/>}
      <div
        class="rfm-marquee"
        style={marqueeStyle}
        onAnimationIteration={onCycleComplete}
        onAnimationEnd={onFinish}
      >
        <div class="rfm-initial-child-container" ref="marqueeRef">
          <div style={childStyle} class="rfm-child">
            {this.renderChild()}
          </div>
        </div>
        {this.renderMultiple(this.multiplier - 1)}
      </div>
      <div class="rfm-marquee" style={marqueeStyle}>
        {this.renderMultiple(this.multiplier)}
      </div>
    </div>
  }
}
</script>

<style lang="scss">
.rfm-marquee-container {
  overflow-x: hidden;
  display: flex;
  flex-direction: row;
  position: relative;
  width: var(--width);
  transform: var(--transform);

  &:hover div {
    animation-play-state: var(--pause-on-hover);
  }

  &:active div {
    animation-play-state: var(--pause-on-click);
  }
}

.rfm-overlay {
  position: absolute;
  width: 100%;
  height: 100%;

  @mixin gradient {
    background: linear-gradient(to right, var(--gradient-color), rgb(255, 255, 255, 0));
  }

  &::before,
  &::after {
    @include gradient;
    content: "";
    height: 100%;
    position: absolute;
    width: var(--gradient-width);
    z-index: 2;
    pointer-events: none;
    touch-action: none;
  }

  &::after {
    right: 0;
    top: 0;
    transform: rotateZ(180deg);
  }

  &::before {
    left: 0;
    top: 0;
  }
}

.rfm-marquee {
  flex: 0 0 auto;
  min-width: var(--min-width);
  z-index: 1;
  display: flex;
  flex-direction: row;
  align-items: center;
  animation: scroll var(--duration) linear var(--delay) var(--iteration-count);
  animation-play-state: var(--play);
  animation-delay: var(--delay);
  animation-direction: var(--direction);

  @keyframes scroll {
    0% {
      transform: translateX(0%);
    }
    100% {
      transform: translateX(-100%);
    }
  }
}

.rfm-initial-child-container {
  flex: 0 0 auto;
  display: flex;
  min-width: auto;
  flex-direction: row;
  align-items: center;
}

.rfm-child {
  transform: var(--transform);
}
</style>
