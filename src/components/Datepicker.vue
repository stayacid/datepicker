<template>
  <div class="date-selector" ref="date">
    <div class="year" ref="year1"></div>
    <div class="month" ref="month1"></div>
    <div class="day" ref="day1"></div>
  </div>
</template>

<script lang="ts">
import { Component, Prop, Vue } from "vue-property-decorator";

const easing = {
  easeOutCubic: function (pos: number) {
    return Math.pow(pos - 1, 3) + 1;
  },
  easeOutQuart: function (pos: number) {
    return -(Math.pow(pos - 1, 4) - 1);
  },
};

class IosSelector {
  constructor(options) {
    let defaults = {
      el: "", // dom
      type: "infinite", // infinite，normal
      count: 20, // Ring specification, the number of options on the ring, must be a multiple of 4
      sensitivity: 0.8,
      source: [], // {value: xx, text: xx}
      value: null,
      onChange: null,
    };

    this.options = Object.assign({}, defaults, options);
    this.options.count = this.options.count * 2 + 2;
    this.options.count = this.options.count - (this.options.count % 4);
    Object.assign(this, this.options); // ?

    this.halfCount = this.options.count / 2;
    this.quarterCount = this.options.count / 4;
    this.a = this.options.sensitivity * 10; // Scroll deceleration
    this.minV = Math.sqrt(1 / this.a); // Minimum initial velocity
    this.selected = this.source[0];

    this.exceedA = 10; // Deceleration exceeded
    this.moveT = 0; // Scroll tick
    this.moving = false;

    this.elems = {
      el: this.refs(this.options.el),
      circleList: null,
      circleItems: null, // list

      highlight: null,
      highlightList: null,
      highListItems: null, // list
    };
    this.events = {
      touchstart: null,
      touchmove: null,
      touchend: null,
    };

    this.itemHeight = (this.elems.el.offsetHeight * 3) / this.options.count; // Each height
    this.itemAngle = 360 / this.options.count; // Degree of rotation between each item
    this.radius = this.itemHeight / Math.tan((this.itemAngle * Math.PI) / 180); //Circle radius. 90 ok

    this.scroll = 0; // The unit is the height of an item (degrees)
    this._init();
  }

  _init() {
    this._create(this.options.source);

    let touchData = {
      startY: 0,
      yArr: [],
    };

    for (let eventName in this.events) {
      this.events[eventName] = ((eventName) => {
        return (e) => {
          if (this.elems.el.contains(e.target) || e.target === this.elems.el) {
            e.preventDefault();
            if (this.source.length) {
              this["_" + eventName](e, touchData);
            }
          }
        };
      })(eventName);
    }

    this.elems.el.addEventListener("touchstart", this.events.touchstart);
    document.addEventListener("mousedown", this.events.touchstart);
    this.elems.el.addEventListener("touchend", this.events.touchend);
    document.addEventListener("mouseup", this.events.touchend);
    if (this.source.length) {
      this.value = this.value !== null ? this.value : this.source[0].value;
      this.select(this.value);
    }
  }

  _touchstart(e, touchData) {
    this.elems.el.addEventListener("touchmove", this.events.touchmove);
    document.addEventListener("mousemove", this.events.touchmove);
    let eventY = e.clientY || e.touches[0].clientY;
    touchData.startY = eventY;
    touchData.yArr = [[eventY, new Date().getTime()]];
    touchData.touchScroll = this.scroll;
    this._stop();
  }

  _touchmove(e, touchData) {
    let eventY = e.clientY || e.touches[0].clientY;
    touchData.yArr.push([eventY, new Date().getTime()]);
    if (touchData.length > 5) {
      touchData.unshift();
    }

    let scrollAdd = (touchData.startY - eventY) / this.itemHeight;
    let moveToScroll = scrollAdd + this.scroll;

    // When scrolling is not infinite, out of range makes scrolling difficult
    if (this.type === "normal") {
      if (moveToScroll < 0) {
        moveToScroll *= 0.3;
      } else if (moveToScroll > this.source.length) {
        moveToScroll =
          this.source.length + (moveToScroll - this.source.length) * 0.3;
      }
      // console.log(moveToScroll);
    } else {
      moveToScroll = this._normalizeScroll(moveToScroll);
    }

    touchData.touchScroll = this._moveTo(moveToScroll);
  }

  _touchend(e, touchData) {
    // console.log(e);
    this.elems.el.removeEventListener("touchmove", this.events.touchmove);
    document.removeEventListener("mousemove", this.events.touchmove);

    let v;

    if (touchData.yArr.length === 1) {
      v = 0;
    } else {
      let startTime = touchData.yArr[touchData.yArr.length - 2][1];
      let endTime = touchData.yArr[touchData.yArr.length - 1][1];
      let startY = touchData.yArr[touchData.yArr.length - 2][0];
      let endY = touchData.yArr[touchData.yArr.length - 1][0];

      // Calculation speed
      v = (((startY - endY) / this.itemHeight) * 1000) / (endTime - startTime);
      let sign = v > 0 ? 1 : -1;

      v = Math.abs(v) > 30 ? 30 * sign : v;
    }

    this.scroll = touchData.touchScroll;
    this._animateMoveByInitV(v);

    // console.log('end');
  }

  _create(source) {
    if (!source.length) {
      return;
    }

    let template = `
      <div class="select-wrap">
        <ul class="select-options" style="transform: translate3d(0, 0, ${-this
          .radius}px) rotateX(0deg);">
          {{circleListHTML}}
          <!-- <li class="select-option">a0</li> -->
        </ul>
        <div class="highlight">
          <ul class="highlight-list">
            <!-- <li class="highlight-item"></li> -->
            {{highListHTML}}
          </ul>
        </div>
      </div>
    `;

    // source processing
    if (this.options.type === "infinite") {
      let concatSource = [].concat(source);
      while (concatSource.length < this.halfCount) {
        concatSource = concatSource.concat(source);
      }
      source = concatSource;
    }
    this.source = source;
    let sourceLength = source.length;

    // Circle HTML
    let circleListHTML = "";
    for (let i = 0; i < source.length; i++) {
      circleListHTML += `<li class="select-option"
                    style="
                      top: ${this.itemHeight * -0.5}px;
                      height: ${this.itemHeight}px;
                      line-height: ${this.itemHeight}px;
                      transform: rotateX(${
                        -this.itemAngle * i
                      }deg) translate3d(0, -0, ${this.radius}px);
                    "
                    data-index="${i}"
                    >${source[i].text}</li>`;
    }

    // Middle highlight HTML
    let highListHTML = "";
    for (let i = 0; i < source.length; i++) {
      highListHTML += `<li class="highlight-item" style="height: ${this.itemHeight}px;">
                        ${source[i].text}
                      </li>`;
    }

    if (this.options.type === "infinite") {
      // Ring head and tail
      for (let i = 0; i < this.quarterCount; i++) {
        // head
        circleListHTML =
          `<li class="select-option"
                      style="
                        top: ${this.itemHeight * -0.5}px;
                        height: ${this.itemHeight}px;
                        line-height: ${this.itemHeight}px;
                        transform: rotateX(${
                          this.itemAngle * (i + 1)
                        }deg) translate3d(0, 0, ${this.radius}px);
                      "
                      data-index="${-i - 1}"
                      >${source[sourceLength - i - 1].text}</li>` +
          circleListHTML;
        // tail
        circleListHTML += `<li class="select-option"
                      style="
                        top: ${this.itemHeight * -0.5}px;
                        height: ${this.itemHeight}px;
                        line-height: ${this.itemHeight}px;
                        transform: rotateX(${
                          -this.itemAngle * (i + sourceLength)
                        }deg) translate3d(0, 0, ${this.radius}px);
                      "
                      data-index="${i + sourceLength}"
                      >${source[i].text}</li>`;
      }

      // Highlight head and tail
      highListHTML =
        `<li class="highlight-item" style="height: ${this.itemHeight}px;">
                          ${source[sourceLength - 1].text}
                      </li>` + highListHTML;
      highListHTML += `<li class="highlight-item" style="height: ${this.itemHeight}px;">${source[0].text}</li>`;
    }

    this.elems.el.innerHTML = template
      .replace("{{circleListHTML}}", circleListHTML)
      .replace("{{highListHTML}}", highListHTML);
    this.elems.circleList = this.elems.el.querySelector(".select-options");
    this.elems.circleItems = this.elems.el.querySelectorAll(".select-option");

    this.elems.highlight = this.elems.el.querySelector(".highlight");
    this.elems.highlightList = this.elems.el.querySelector(".highlight-list");
    this.elems.highlightitems = this.elems.el.querySelectorAll(
      ".highlight-item"
    );

    if (this.type === "infinite") {
      this.elems.highlightList.style.top = -this.itemHeight + "px";
    }

    this.elems.highlight.style.height = this.itemHeight + "px";
    this.elems.highlight.style.lineHeight = this.itemHeight + "px";
  }

  /**
   * Modulo scroll，eg source.length = 5 scroll = 6.1
   * After taking the modulus normalizedScroll = 1.1
   * @param {init} scroll
   * @return After modulo normalizedScroll
   */
  _normalizeScroll(scroll) {
    let normalizedScroll = scroll;

    while (normalizedScroll < 0) {
      normalizedScroll += this.source.length;
    }
    normalizedScroll = normalizedScroll % this.source.length;
    return normalizedScroll;
  }

  /**
   * Position to scroll, no animation
   * @param {init} scroll
   * @return Return the scroll after the specified normalize
   */
  _moveTo(scroll) {
    if (this.type === "infinite") {
      scroll = this._normalizeScroll(scroll);
    }
    this.elems.circleList.style.transform = `translate3d(0, 0, ${-this
      .radius}px) rotateX(${this.itemAngle * scroll}deg)`;
    this.elems.highlightList.style.transform = `translate3d(0, ${
      -scroll * this.itemHeight
    }px, 0)`;

    [...this.elems.circleItems].forEach((itemElem) => {
      if (Math.abs(itemElem.dataset.index - scroll) > this.quarterCount) {
        itemElem.style.visibility = "hidden";
      } else {
        itemElem.style.visibility = "visible";
      }
    });

    // console.log(scroll);
    // console.log(`translate3d(0, 0, ${-this.radius}px) rotateX(${-this.itemAngle * scroll}deg)`);
    return scroll;
  }

  /**
   * Scroll at initial speed initV
   * @param {init} initV， initV will be reset
   * To ensure scrolling to the integer scroll according to the acceleration (guaranteed to be able to locate a selected value by scroll)
   */
  async _animateMoveByInitV(initV) {
    // console.log(initV);

    let initScroll;
    let finalScroll;
    let finalV;

    let totalScrollLen;
    let a;
    let t;

    if (this.type === "normal") {
      if (this.scroll < 0 || this.scroll > this.source.length - 1) {
        a = this.exceedA;
        initScroll = this.scroll;
        finalScroll = this.scroll < 0 ? 0 : this.source.length - 1;
        totalScrollLen = initScroll - finalScroll;

        t = Math.sqrt(Math.abs(totalScrollLen / a));
        initV = a * t;
        initV = this.scroll > 0 ? -initV : initV;
        finalV = 0;
        await this._animateToScroll(initScroll, finalScroll, t);
      } else {
        initScroll = this.scroll;
        a = initV > 0 ? -this.a : this.a; // Deceleration acceleration
        t = Math.abs(initV / a); // Speed reduced to 0 takes time
        totalScrollLen = initV * t + (a * t * t) / 2; // Total rolling length
        finalScroll = Math.round(this.scroll + totalScrollLen); // Round to ensure accuracy and finally scroll to an integer
        finalScroll =
          finalScroll < 0
            ? 0
            : finalScroll > this.source.length - 1
            ? this.source.length - 1
            : finalScroll;

        totalScrollLen = finalScroll - initScroll;
        t = Math.sqrt(Math.abs(totalScrollLen / a));
        await this._animateToScroll(
          this.scroll,
          finalScroll,
          t,
          "easeOutQuart"
        );
      }
    } else {
      initScroll = this.scroll;

      a = initV > 0 ? -this.a : this.a; // Deceleration acceleration
      t = Math.abs(initV / a); // Speed reduced to 0 takes time
      totalScrollLen = initV * t + (a * t * t) / 2; // Total rolling length
      finalScroll = Math.round(this.scroll + totalScrollLen); // Round to ensure accuracy and finally scroll to an integer
      await this._animateToScroll(this.scroll, finalScroll, t, "easeOutQuart");
    }

    // await this._animateToScroll(this.scroll, finalScroll, initV, 0);

    this._selectByScroll(this.scroll);
  }

  _animateToScroll(initScroll, finalScroll, t, easingName = "easeOutQuart") {
    if (initScroll === finalScroll || t === 0) {
      this._moveTo(initScroll);
      return;
    }

    let start = new Date().getTime() / 1000;
    let pass = 0;
    let totalScrollLen = finalScroll - initScroll;

    // console.log(initScroll, finalScroll, initV, finalV, a);
    return new Promise((resolve, reject) => {
      this.moving = true;
      let tick = () => {
        pass = new Date().getTime() / 1000 - start;

        if (pass < t) {
          this.scroll = this._moveTo(
            initScroll + easing[easingName](pass / t) * totalScrollLen
          );
          this.moveT = requestAnimationFrame(tick);
        } else {
          resolve();
          this._stop();
          this.scroll = this._moveTo(initScroll + totalScrollLen);
        }
      };
      tick();
    });
  }

  _stop() {
    this.moving = false;
    cancelAnimationFrame(this.moveT);
  }

  _selectByScroll(scroll) {
    scroll = this._normalizeScroll(scroll) | 0;
    if (scroll > this.source.length - 1) {
      scroll = this.source.length - 1;
      this._moveTo(scroll);
    }
    this._moveTo(scroll);
    this.scroll = scroll;
    this.selected = this.source[scroll];
    this.value = this.selected.value;
    this.onChange && this.onChange(this.selected);
  }

  updateSource(source) {
    this._create(source);

    if (!this.moving) {
      this._selectByScroll(this.scroll);
    }
  }

  select(value) {
    for (let i = 0; i < this.source.length; i++) {
      if (this.source[i].value === value) {
        window.cancelAnimationFrame(this.moveT);
        // this.scroll = this._moveTo(i);
        let initScroll = this._normalizeScroll(this.scroll);
        let finalScroll = i;
        let t = Math.sqrt(Math.abs((finalScroll - initScroll) / this.a));
        this._animateToScroll(initScroll, finalScroll, t);
        setTimeout(() => this._selectByScroll(i));
        return;
      }
    }
    throw new Error(
      `can not select value: ${value}, ${value} match nothing in current source`
    );
  }

  destroy() {
    this._stop();
    // document event unbinding
    for (let eventName in this.events) {
      this.elems.el.removeEventListener("eventName", this.events[eventName]);
    }
    document.removeEventListener("mousedown", this.events["touchstart"]);
    document.removeEventListener("mousemove", this.events["touchmove"]);
    document.removeEventListener("mouseup", this.events["touchend"]);
    // Element removal
    this.elems.el.innerHTML = "";
    this.elems = null;
  }
}

@Component
export default class HelloWorld extends Vue {
  //@Prop() private msg!: string;
  // data
  // save current date
  currentYear = new Date().getFullYear();
  currentMonth = 1;
  currentDay = 1;
  // fill archives with dates
  yearSource = this.getYears();
  monthSource = this.getMonths();
  daySource = this.getDays(this.currentYear, this.currentMonth);
  //
  yearSelector = new IosSelector({
    el: "year1",
    type: "infinite",
    source: this.yearSource,
    count: 3,
    onChange: (selected) => {
      this.currentYear = selected.value;
      this.daySource = this.getDays(this.currentYear, this.currentMonth);
      this.daySelector.updateSource(this.daySource);
      console.log(
        this.yearSelector.value,
        this.monthSelector.value,
        this.daySelector.value
      );
    },
  });

  monthSelector = new IosSelector({
    el: "month1",
    type: "infinite",
    source: this.monthSource,
    count: 3,
    onChange: (selected) => {
      this.currentMonth = selected.value;

      this.daySource = this.getDays(this.currentYear, this.currentMonth);
      this.daySelector.updateSource(this.daySource);
      console.log(
        this.yearSelector.value,
        this.monthSelector.value,
        this.daySelector.value
      );
    },
  });

  daySelector = new IosSelector({
    el: "day1",
    type: "infinite",
    source: [],
    count: 3,
    onChange: (selected) => {
      this.currentDay = selected.value;
      console.log(
        this.yearSelector.value,
        this.monthSelector.value,
        this.daySelector.value
      );
    },
  });

  //methods:
  // date logic

  // return archive with years
  getYears() {
    let currentYear = new Date().getFullYear();
    let years = [];

    for (let i = currentYear - 20; i < currentYear + 20; i++) {
      years.push({
        value: i,
        text: i,
      });
    }
    return years;
  }

  // return archive with months
  getMonths() {
    let months = [];
    for (let i = 1; i <= 12; i++) {
      months.push({
        value: i,
        text: i,
      });
    }
    return months;
  }

  // return archive with days
  getDays(year: number, month: number) {
    let dayCount = new Date(year, month, 0).getDate();
    let days = [];

    for (let i = 1; i <= dayCount; i++) {
      days.push({
        value: i,
        text: i,
      });
    }

    return days;
  }

  // hook
  created() {
    // create year ring

    // create months ring

    // create days ring

    let now = new Date();

    setTimeout(function() {
      this.yearSelector.select(now.getFullYear());
      this.monthSelector.select(now.getMonth() + 1);
      this.daySelector.select(now.getDate());
    }, 100);
  }
}
</script>

<style scoped lang="scss">
.select-wrap {
  position: relative;
  // top: 200px;
  height: 100%;
  // perspective: 1200px;
  text-align: center;
  overflow: hidden;
  font-size: 20px;
  color: #fff;

  /*&:before,
  &:after {
    position: absolute;
    z-index: 1;
    display: block;
    content: "";
    width: 100%;
    height: 50%;
  }
  
  &:before {
    top: 0;
    background-image: linear-gradient(
      to bottom,
      rgba(1, 1, 1, 0.5),
      rgba(1, 1, 1, 0)
    );
  }
  &:after {
    bottom: 0;
    background-image: linear-gradient(
      to top,
      rgba(1, 1, 1, 0.5),
      rgba(1, 1, 1, 0)
    );
  }*/

  .select-options {
    position: absolute;
    top: 50%;
    left: 0;
    width: 100%;
    height: 0;
    transform-style: preserve-3d;
    margin: 0 auto;
    display: block;
    transform: translateZ(-150px) rotateX(0deg);
    -webkit-font-smoothing: subpixel-antialiased;
    color: #fff;

    .select-option {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 50px;
      -webkit-font-smoothing: subpixel-antialiased;

      @for $i from 1 through 100 {
        &:nth-child(#{$i}) {
          //transform: rotateX(-15deg * ($i - 1)) translateZ(150px);
          transform: translateY(-100px);
        }
      }
    }
  }
}

.highlight {
  position: absolute;
  top: 50%;
  transform: translate(0, -50%);
  width: 100%;
  background-color: #000;
  border-top: 1px solid #333;
  border-bottom: 1px solid #333;
  font-size: 24px;
  overflow: hidden;
}
.highlight-list {
  // display: none;
  position: absolute;
  width: 100%;
}

/* date */
.date-selector {
  position: absolute;
  left: 50%;
  top: 50%;
  transform: translate(-50%, -50%);
  //perspective: 2000px;
  display: flex;
  align-items: stretch;
  justify-content: space-between;
  width: 600px;
  height: 300px;

  > div {
    flex: 1;
    max-width: 30%;
  }
}
</style>
