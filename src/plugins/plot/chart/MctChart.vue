<!--
 Open MCT, Copyright (c) 2014-2023, United States Government
 as represented by the Administrator of the National Aeronautics and Space
 Administration. All rights reserved.

 Open MCT is licensed under the Apache License, Version 2.0 (the
 "License"); you may not use this file except in compliance with the License.
 You may obtain a copy of the License at
 http://www.apache.org/licenses/LICENSE-2.0.

 Unless required by applicable law or agreed to in writing, software
 distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
 WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
 License for the specific language governing permissions and limitations
 under the License.

 Open MCT includes source code licensed under additional open source
 licenses. See the Open Source Licenses file (LICENSES.md) included with
 this source code distribution or the Licensing information page available
 at runtime from the About dialog for additional information.
-->

<template>
  <div ref="chart" class="gl-plot-chart-area">
    <canvas :style="canvasStyle" class="js-overlay-canvas"></canvas>
    <canvas :style="canvasStyle" class="js-main-canvas"></canvas>
    <div ref="limitArea" class="js-limit-area">
      <limit-label
        v-for="(limitLabel, index) in visibleLimitLabels"
        :key="index"
        :point="limitLabel.point"
        :limit="limitLabel.limit"
      ></limit-label>
      <limit-line
        v-for="(limitLine, index) in visibleLimitLines"
        :key="index"
        :point="limitLine.point"
        :limit="limitLine.limit"
      ></limit-line>
    </div>
  </div>
</template>

<script>
import mount from 'utils/mount';
import { toRaw } from 'vue';

import configStore from '../configuration/ConfigStore';
import PlotConfigurationModel from '../configuration/PlotConfigurationModel';
import { DrawLoader } from '../draw/DrawLoader';
import eventHelpers from '../lib/eventHelpers';
import LimitLabel from './LimitLabel.vue';
import LimitLine from './LimitLine.vue';
import MCTChartAlarmLineSet from './MCTChartAlarmLineSet';
import MCTChartAlarmPointSet from './MCTChartAlarmPointSet';
import MCTChartLineLinear from './MCTChartLineLinear';
import MCTChartLineStepAfter from './MCTChartLineStepAfter';
import MCTChartPointSet from './MCTChartPointSet';

const MARKER_SIZE = 6.0;
const HIGHLIGHT_SIZE = MARKER_SIZE * 2.0;
const ANNOTATION_SIZE = MARKER_SIZE * 3.0;
const CLEARANCE = 15;
// These attributes are changed in the plot model, but we don't need to react to the changes.
const NO_HANDLING_NEEDED_ATTRIBUTES = {
  label: 'label',
  values: 'values',
  format: 'format',
  color: 'color',
  name: 'name',
  unit: 'unit'
};
// These attributes in turn set one of HANDLED_ATTRIBUTES, so we don't need specific listeners for them - this prevents excessive redraws.
const IMPLICIT_HANDLED_ATTRIBUTES = {
  range: 'range',
  //series stats update y axis stats
  stats: 'stats',
  frozen: 'frozen',
  autoscale: 'autoscale',
  autoscalePadding: 'autoscalePadding',
  logMode: 'logMode',
  yKey: 'yKey'
};
// Attribute changes that we are specifically handling with listeners
const HANDLED_ATTRIBUTES = {
  //X and Y Axis attributes
  key: 'key',
  displayRange: 'displayRange',
  //series attributes
  xKey: 'xKey',
  interpolate: 'interpolate',
  markers: 'markers',
  markerShape: 'markerShape',
  markerSize: 'markerSize',
  alarmMarkers: 'alarmMarkers',
  limitLines: 'limitLines',
  yAxisId: 'yAxisId'
};

export default {
  components: { LimitLine, LimitLabel },
  inject: ['openmct', 'domainObject', 'path', 'renderWhenVisible'],
  props: {
    rectangles: {
      type: Array,
      default() {
        return [];
      }
    },
    highlights: {
      type: Array,
      default() {
        return [];
      }
    },
    annotatedPointsBySeries: {
      type: Object,
      default() {
        return {};
      }
    },
    annotationSelectionsBySeries: {
      type: Object,
      default() {
        return {};
      }
    },
    showLimitLineLabels: {
      type: Object,
      default() {
        return undefined;
      }
    },
    hiddenYAxisIds: {
      type: Array,
      default() {
        return [];
      }
    },
    annotationViewingAndEditingAllowed: {
      type: Boolean,
      required: true
    }
  },
  emits: ['chart-loaded', 'plot-reinitialize-canvas'],
  data() {
    return {
      visibleLimitLabels: [],
      visibleLimitLines: []
    };
  },
  computed: {
    canvasStyle() {
      return {
        position: 'absolute',
        background: 'none',
        width: '100%',
        height: '100%'
      };
    }
  },
  watch: {
    highlights: {
      handler() {
        this.scheduleDraw();
      },
      deep: true
    },
    annotatedPointsBySeries: {
      handler() {
        this.scheduleDraw();
      }
    },
    annotationSelectionsBySeries: {
      handler() {
        this.scheduleDraw();
      }
    },
    rectangles: {
      handler() {
        this.scheduleDraw();
      },
      deep: true
    },
    showLimitLineLabels() {
      this.updateLimitLines();
    },
    hiddenYAxisIds: {
      handler() {
        this.hiddenYAxisIds.forEach((id) => {
          this.resetYOffsetAndSeriesDataForYAxis(id);
          this.updateLimitLines();
        });
        this.scheduleDraw();
      },
      deep: true
    }
  },
  mounted() {
    this.chartVisible = true;
    this.chartContainer = this.$refs.chart;
    this.drawnOnce = false;
    this.visibilityObserver = new IntersectionObserver(this.visibilityChanged);
    eventHelpers.extend(this);
    this.seriesModels = [];
    this.config = this.getConfig();
    this.isDestroyed = false;
    this.lines = [];
    this.limitLines = [];
    this.pointSets = [];
    this.alarmSets = [];
    const yAxisId = this.config.yAxis.get('id');
    this.offset = {
      [yAxisId]: {}
    };
    this.listenTo(
      this.config.yAxis,
      `change:${HANDLED_ATTRIBUTES.displayRange}`,
      this.scheduleDraw
    );
    this.listenTo(
      this.config.yAxis,
      `change:${HANDLED_ATTRIBUTES.key}`,
      this.resetYOffsetAndSeriesDataForYAxis.bind(this, yAxisId),
      this
    );
    this.listenTo(this.config.yAxis, 'change', this.redrawIfNotAlreadyHandled);
    if (this.config.additionalYAxes.length) {
      this.config.additionalYAxes.forEach((yAxis) => {
        const id = yAxis.get('id');
        this.offset[id] = {};
        this.listenTo(yAxis, `change:${HANDLED_ATTRIBUTES.displayRange}`, this.scheduleDraw);
        this.listenTo(
          yAxis,
          `change:${HANDLED_ATTRIBUTES.key}`,
          this.resetYOffsetAndSeriesDataForYAxis.bind(this, id),
          this
        );
        this.listenTo(yAxis, 'change', this.redrawIfNotAlreadyHandled);
      });
    }

    this.seriesElements = new WeakMap();
    this.seriesLimits = new WeakMap();

    const canvasReadyForDrawing = this.readyCanvasForDrawing();
    if (canvasReadyForDrawing) {
      this.draw();
    }

    this.listenTo(this.config.series, 'add', this.onSeriesAdd, this);
    this.listenTo(this.config.series, 'remove', this.onSeriesRemove, this);

    this.listenTo(this.config.xAxis, 'change:displayRange', this.scheduleDraw);
    this.listenTo(this.config.xAxis, 'change', this.redrawIfNotAlreadyHandled);
    this.config.series.forEach(this.onSeriesAdd, this);
    this.$emit('chart-loaded');
  },
  beforeUnmount() {
    this.destroy();
    this.visibilityObserver.unobserve(this.chartContainer);
  },
  methods: {
    getConfig() {
      const configId = this.openmct.objects.makeKeyString(this.domainObject.identifier);
      let config = configStore.get(configId);
      if (!config) {
        config = new PlotConfigurationModel({
          id: configId,
          domainObject: this.domainObject,
          openmct: this.openmct
        });
        configStore.add(configId, config);
      }

      return config;
    },
    visibilityChanged([entry]) {
      if (entry.target === this.chartContainer) {
        const wasVisible = this.chartVisible;
        this.chartVisible = entry.isIntersecting;
        if (!this.chartVisible) {
          // destroy the chart
          this.destroyCanvas();
        } else if (!wasVisible && this.chartVisible) {
          // rebuild the chart
          this.buildCanvasElements();
          const canvasInitialized = this.readyCanvasForDrawing();
          if (canvasInitialized) {
            this.draw();
          }
          this.$emit('plot-reinitialize-canvas');
        } else if (wasVisible && this.chartVisible) {
          // ignore, moving on
        }
      }
    },
    reDraw(newXKey, oldXKey, series) {
      this.changeInterpolate(newXKey, oldXKey, series);
      this.changeMarkers(newXKey, oldXKey, series);
      this.changeAlarmMarkers(newXKey, oldXKey, series);
      this.changeLimitLines(newXKey, oldXKey, series);
    },
    onSeriesAdd(series, index) {
      this.seriesModels[index] = series;
      this.listenTo(series, `change:${HANDLED_ATTRIBUTES.xKey}`, this.reDraw, this);
      this.listenTo(
        series,
        `change:${HANDLED_ATTRIBUTES.interpolate}`,
        this.changeInterpolate,
        this
      );
      this.listenTo(series, `change:${HANDLED_ATTRIBUTES.markers}`, this.changeMarkers, this);
      this.listenTo(
        series,
        `change:${HANDLED_ATTRIBUTES.alarmMarkers}`,
        this.changeAlarmMarkers,
        this
      );
      this.listenTo(series, `change:${HANDLED_ATTRIBUTES.limitLines}`, this.changeLimitLines, this);
      this.listenTo(series, `change:${HANDLED_ATTRIBUTES.yAxisId}`, this.resetAxisAndRedraw, this);
      this.listenTo(series, `change:${HANDLED_ATTRIBUTES.markerShape}`, this.scheduleDraw, this);
      this.listenTo(series, `change:${HANDLED_ATTRIBUTES.markerSize}`, this.scheduleDraw, this);
      this.listenTo(series, 'change', this.redrawIfNotAlreadyHandled);
      this.listenTo(series, 'add', this.onAddPoint);
      this.makeChartElement(series);
      this.makeLimitLines(series);
    },
    onSeriesRemove(seriesToRemove) {
      this.stopListening(seriesToRemove);
      this.removeChartElement(seriesToRemove);
      this.scheduleDraw();

      const seriesIndexToRemove = this.seriesModels.findIndex(
        (series) => series.keyString === seriesToRemove.keyString
      );
      this.seriesModels.splice(seriesIndexToRemove, 1);
    },
    onAddPoint(point, insertIndex, series) {
      const mainYAxisId = this.config.yAxis.get('id');
      const seriesYAxisId = series.get('yAxisId');
      const xRange = this.config.xAxis.get('displayRange');

      let yRange;
      if (seriesYAxisId === mainYAxisId) {
        yRange = this.config.yAxis.get('displayRange');
      } else {
        yRange = this.config.additionalYAxes
          .find((yAxis) => yAxis.get('id') === seriesYAxisId)
          .get('displayRange');
      }

      const xValue = series.getXVal(point);
      const yValue = series.getYVal(point);

      // if user is not looking at data within the current bounds, don't draw the point
      if (
        xValue > xRange.min &&
        xValue < xRange.max &&
        yValue > yRange.min &&
        yValue < yRange.max
      ) {
        this.scheduleDraw();
      }
    },
    changeInterpolate(mode, o, series) {
      if (mode === o) {
        return;
      }

      const elements = this.seriesElements.get(toRaw(series));
      elements.lines.forEach(function (line) {
        this.lines.splice(this.lines.indexOf(line), 1);
        line.destroy();
      }, this);
      elements.lines = [];

      const newLine = this.lineForSeries(series);
      if (newLine) {
        elements.lines.push(newLine);
        this.lines.push(newLine);
      }
    },
    changeAlarmMarkers(mode, o, series) {
      if (mode === o) {
        return;
      }

      const elements = this.seriesElements.get(toRaw(series));
      if (elements.alarmSet) {
        elements.alarmSet.destroy();
        this.alarmSets.splice(this.alarmSets.indexOf(elements.alarmSet), 1);
      }

      elements.alarmSet = this.alarmPointSetForSeries(series);
      if (elements.alarmSet) {
        this.alarmSets.push(elements.alarmSet);
      }
    },
    changeMarkers(mode, o, series) {
      if (mode === o) {
        return;
      }

      const elements = this.seriesElements.get(toRaw(series));
      elements.pointSets.forEach(function (pointSet) {
        this.pointSets.splice(this.pointSets.indexOf(pointSet), 1);
        pointSet.destroy();
      }, this);
      elements.pointSets = [];

      const pointSet = this.pointSetForSeries(series);
      if (pointSet) {
        elements.pointSets.push(pointSet);
        this.pointSets.push(pointSet);
      }
    },
    changeLimitLines(showLimitLines, oldShowLimitLines, series) {
      if (showLimitLines === oldShowLimitLines) {
        return;
      }

      this.makeLimitLines(series);
      this.updateLimitLines();
      this.scheduleDraw();
    },
    resetAxisAndRedraw(newYAxisId, oldYAxisId, series) {
      if (!oldYAxisId) {
        return;
      }

      //Remove the old chart elements for the series since their offsets are pointing to the old y axis
      this.removeChartElement(series);
      this.resetYOffsetAndSeriesDataForYAxis(oldYAxisId);

      //Make the chart elements again for the new y-axis and offset
      this.makeChartElement(series);
      this.makeLimitLines(series);
      this.updateLimitLines();
      this.scheduleDraw();
    },
    destroy() {
      this.destroyCanvas();
      this.isDestroyed = true;
      this.lines.forEach((line) => line.destroy());
      this.limitLines.forEach((line) => line.destroy());
      this.pointSets.forEach((pointSet) => pointSet.destroy());
      this.alarmSets.forEach((alarmSet) => alarmSet.destroy());
    },
    resetYOffsetAndSeriesDataForYAxis(yAxisId) {
      delete this.offset[yAxisId].y;
      delete this.offset[yAxisId].xVal;
      delete this.offset[yAxisId].yVal;
      delete this.offset[yAxisId].xKey;
      delete this.offset[yAxisId].yKey;

      this.resetResetChartElements(yAxisId);
    },
    resetResetChartElements(yAxisId) {
      const lines = this.lines.filter(this.matchByYAxisIdExcludingVisibility.bind(this, yAxisId));
      lines.forEach(function (line) {
        line.reset();
      });
      const limitLines = this.limitLines.filter(
        this.matchByYAxisIdExcludingVisibility.bind(this, yAxisId)
      );
      limitLines.forEach(function (line) {
        line.reset();
      });
      const pointSets = this.pointSets.filter(
        this.matchByYAxisIdExcludingVisibility.bind(this, yAxisId)
      );
      pointSets.forEach(function (pointSet) {
        pointSet.reset();
      });
    },
    setOffset(offsetPoint, index, series) {
      const mainYAxisId = this.config.yAxis.get('id');
      const yAxisId = series.get('yAxisId') || mainYAxisId;
      if (this.offset[yAxisId].x && this.offset[yAxisId].y) {
        return;
      }

      const offsets = {
        x: series.getXVal(offsetPoint),
        y: series.getYVal(offsetPoint)
      };

      this.offset[yAxisId].x = function (x) {
        return x - offsets.x;
      }.bind(this);
      this.offset[yAxisId].y = function (y) {
        return y - offsets.y;
      }.bind(this);
      this.offset[yAxisId].xVal = function (point, pSeries) {
        return this.offset[yAxisId].x(pSeries.getXVal(point));
      }.bind(this);
      this.offset[yAxisId].yVal = function (point, pSeries) {
        return this.offset[yAxisId].y(pSeries.getYVal(point));
      }.bind(this);
    },
    destroyCanvas() {
      if (this.isDestroyed) {
        return;
      }
      this.stopListening(this.drawAPI);
      DrawLoader.releaseDrawAPI(this.drawAPI);
      if (this.chartContainer) {
        const canvasElements = this.chartContainer.querySelectorAll('canvas');
        canvasElements.forEach((canvas) => {
          canvas.parentNode.removeChild(canvas);
        });
      }
    },
    readyCanvasForDrawing() {
      const canvasEls = this.chartContainer.querySelectorAll('canvas');
      const mainCanvas = canvasEls[1];
      const overlayCanvas = canvasEls[0];
      this.canvas = mainCanvas;
      this.overlay = overlayCanvas;
      this.drawAPI = DrawLoader.getDrawAPI(mainCanvas, overlayCanvas);
      if (this.drawAPI) {
        this.listenTo(this.drawAPI, 'error', this.fallbackToCanvas, this);
      }

      return Boolean(this.drawAPI);
    },
    buildCanvasElements() {
      const div = document.createElement('div');
      div.innerHTML = `
      <canvas style="position: absolute; background: none; width: 100%; height: 100%;" class="js-overlay-canvas"></canvas>
      <canvas style="position: absolute; background: none; width: 100%; height: 100%;" class="js-main-canvas"></canvas>
      `;
      const mainCanvas = div.querySelectorAll('canvas')[1];
      const overlayCanvas = div.querySelectorAll('canvas')[0];
      this.chartContainer.appendChild(mainCanvas, this.canvas);
      this.canvas = mainCanvas;
      this.chartContainer.appendChild(overlayCanvas, this.overlay);
      this.overlay = overlayCanvas;
    },
    fallbackToCanvas() {
      console.warn(`📈 fallback to 2D canvas`);
      this.destroyCanvas();
      this.buildCanvasElements();
      this.drawAPI = DrawLoader.getFallbackDrawAPI(this.canvas, this.overlay);
      this.$emit('plot-reinitialize-canvas');
    },
    removeChartElement(series) {
      const elements = this.seriesElements.get(toRaw(series));

      elements.lines.forEach(function (line) {
        this.lines.splice(this.lines.indexOf(line), 1);
        line.destroy();
      }, this);
      elements.pointSets.forEach(function (pointSet) {
        this.pointSets.splice(this.pointSets.indexOf(pointSet), 1);
        pointSet.destroy();
      }, this);
      if (elements.alarmSet) {
        elements.alarmSet.destroy();
        this.alarmSets.splice(this.alarmSets.indexOf(elements.alarmSet), 1);
      }

      this.seriesElements.delete(toRaw(series));

      this.clearLimitLines(series);
    },
    lineForSeries(series) {
      const mainYAxisId = this.config.yAxis.get('id');
      const yAxisId = series.get('yAxisId') || mainYAxisId;
      let offset = this.offset[yAxisId];

      if (series.get('interpolate') === 'linear') {
        return new MCTChartLineLinear(series, this, offset);
      }

      if (series.get('interpolate') === 'stepAfter') {
        return new MCTChartLineStepAfter(series, this, offset);
      }
    },
    limitLineForSeries(series) {
      const mainYAxisId = this.config.yAxis.get('id');
      const yAxisId = series.get('yAxisId') || mainYAxisId;
      let offset = this.offset[yAxisId];

      return new MCTChartAlarmLineSet(series, this, offset, this.openmct.time.bounds());
    },
    pointSetForSeries(series) {
      const mainYAxisId = this.config.yAxis.get('id');
      const yAxisId = series.get('yAxisId') || mainYAxisId;
      let offset = this.offset[yAxisId];

      if (series.get('markers')) {
        return new MCTChartPointSet(series, this, offset);
      }
    },
    alarmPointSetForSeries(series) {
      const mainYAxisId = this.config.yAxis.get('id');
      const yAxisId = series.get('yAxisId') || mainYAxisId;
      let offset = this.offset[yAxisId];

      if (series.get('alarmMarkers')) {
        return new MCTChartAlarmPointSet(series, this, offset);
      }
    },
    makeChartElement(series) {
      const elements = {
        lines: [],
        pointSets: [],
        limitLines: []
      };

      const line = this.lineForSeries(series);
      if (line) {
        elements.lines.push(line);
        this.lines.push(line);
      }

      const pointSet = this.pointSetForSeries(series);
      if (pointSet) {
        elements.pointSets.push(pointSet);
        this.pointSets.push(pointSet);
      }

      elements.alarmSet = this.alarmPointSetForSeries(series);
      if (elements.alarmSet) {
        this.alarmSets.push(elements.alarmSet);
      }

      this.seriesElements.set(toRaw(series), elements);
    },
    makeLimitLines(series) {
      this.clearLimitLines(series);

      if (!series || !series.get('limitLines')) {
        return;
      }

      const limitElements = {
        limitLines: []
      };

      const limitLine = this.limitLineForSeries(series);
      if (limitLine) {
        limitElements.limitLines.push(limitLine);
        this.limitLines.push(limitLine);
      }

      this.seriesLimits.set(toRaw(series), limitElements);
    },
    clearLimitLines(series) {
      const seriesLimits = this.seriesLimits.get(toRaw(series));

      if (seriesLimits) {
        seriesLimits.limitLines.forEach(function (line) {
          this.limitLines.splice(this.limitLines.indexOf(line), 1);
          line.destroy();
        }, this);

        this.seriesLimits.delete(toRaw(series));
      }
    },
    canDraw(yAxisId) {
      if (!this.offset[yAxisId] || !this.offset[yAxisId].x || !this.offset[yAxisId].y) {
        return false;
      }

      return true;
    },
    redrawIfNotAlreadyHandled(attribute, value, oldValue) {
      if (Object.keys(HANDLED_ATTRIBUTES).includes(attribute) && oldValue) {
        return;
      }

      if (Object.keys(IMPLICIT_HANDLED_ATTRIBUTES).includes(attribute) && oldValue) {
        return;
      }

      if (Object.keys(NO_HANDLING_NEEDED_ATTRIBUTES).includes(attribute) && oldValue) {
        return;
      }

      this.updateLimitLines();
      this.scheduleDraw();
    },
    scheduleDraw() {
      if (!this.drawScheduled) {
        const called = this.renderWhenVisible(this.draw);
        this.drawScheduled = called;
        if (!this.drawnOnce && called) {
          this.drawnOnce = true;
          this.visibilityObserver.observe(this.chartContainer);
        }
      }
    },
    draw() {
      this.drawScheduled = false;
      if (this.isDestroyed || !this.chartVisible) {
        return;
      }

      this.drawAPI.clear();
      const mainYAxisId = this.config.yAxis.get('id');
      //There has to be at least one yAxis
      const yAxisIds = [mainYAxisId].concat(
        this.config.additionalYAxes.map((yAxis) => yAxis.get('id'))
      );

      // Repeat drawing for all yAxes
      yAxisIds.filter(this.canDraw).forEach((id, yAxisIndex) => {
        this.updateViewport(id);
        this.drawSeries(id);
        if (yAxisIndex === 0) {
          this.drawRectangles(id);
        }

        this.drawHighlights(id);
        // only draw these in fixed time mode or plot is paused
        if (this.annotationViewingAndEditingAllowed) {
          this.prepareToDrawAnnotatedPoints(id);
          this.prepareToDrawAnnotationSelections(id);
        }
      });
    },
    updateViewport(yAxisId) {
      if (!this.chartVisible) {
        return;
      }
      const mainYAxisId = this.config.yAxis.get('id');
      const xRange = this.config.xAxis.get('displayRange');
      let yRange;
      if (yAxisId === mainYAxisId) {
        yRange = this.config.yAxis.get('displayRange');
      } else {
        if (this.config.additionalYAxes.length) {
          const yAxisForId = this.config.additionalYAxes.find(
            (yAxis) => yAxis.get('id') === yAxisId
          );
          yRange = yAxisForId.get('displayRange');
        }
      }

      if (!xRange || !yRange) {
        return;
      }

      const dimensions = [xRange.max - xRange.min, yRange.max - yRange.min];

      let origin;
      origin = [this.offset[yAxisId].x(xRange.min), this.offset[yAxisId].y(yRange.min)];

      this.drawAPI.setDimensions(dimensions, origin);
    },
    // match items by their yAxisId, but don't care if the series is hidden or not.
    matchByYAxisIdExcludingVisibility() {
      const args = Array.from(arguments).slice(0, 4);

      return this.matchByYAxisId(...args, true);
    },
    matchByYAxisId(id, item, index, items, excludeVisibility = false) {
      const mainYAxisId = this.config.yAxis.get('id');
      let matchesId = false;
      const axisSeriesAreVisible = excludeVisibility || this.hiddenYAxisIds.indexOf(id) < 0;
      const series = item.series;
      if (axisSeriesAreVisible && series) {
        const seriesYAxisId = series.get('yAxisId') || mainYAxisId;
        matchesId = seriesYAxisId === id;
      }

      return matchesId;
    },
    drawSeries(id) {
      const lines = this.lines.filter(this.matchByYAxisId.bind(this, id));
      lines.forEach(this.drawLine, this);
      const pointSets = this.pointSets.filter(this.matchByYAxisId.bind(this, id));
      pointSets.forEach(this.drawPoints, this);
      const alarmSets = this.alarmSets.filter(this.matchByYAxisId.bind(this, id));
      alarmSets.forEach(this.drawAlarmPoints, this);
      //console.timeEnd('📈 drawSeries');
    },
    updateLimitLines() {
      this.config.series.models.forEach((series) => {
        const yAxisId = series.get('yAxisId');

        if (this.hiddenYAxisIds.indexOf(yAxisId) < 0) {
          this.updateLimitLinesForSeries(yAxisId, series);
        }
      });
    },
    updateLimitLinesForSeries(yAxisId, series) {
      if (!this.canDraw(yAxisId)) {
        return;
      }

      this.updateViewport(yAxisId);

      if (!this.drawAPI.origin) {
        return;
      }

      let limitPointOverlap = [];
      //reset
      this.visibleLimitLabels = [];
      this.visibleLimitLines = [];

      this.limitLines.forEach((limitLine) => {
        limitLine.limits.forEach((limit) => {
          if (series.keyString !== limit.seriesKey) {
            return;
          }

          const showLabels = this.showLabels(limit.seriesKey);
          if (showLabels) {
            const overlap = this.getLimitOverlap(limit, limitPointOverlap);
            limitPointOverlap.push(overlap);
            this.visibleLimitLabels.push(this.getLimitProps(limit, overlap));
          }

          this.visibleLimitLines.push(this.getLimitElementProps(limit));
        }, this);
      });
    },
    showLabels(seriesKey) {
      return this.showLimitLineLabels?.seriesKey === seriesKey;
    },
    getLimitElementProps(limit) {
      let point = {
        left: 0,
        top: this.drawAPI.y(limit.point.y)
      };

      return {
        point,
        limit
      };
    },
    getLimitElement(limit) {
      let point = {
        left: 0,
        top: this.drawAPI.y(limit.point.y)
      };
      const { vNode, destroy } = mount(LimitLine, {
        props: {
          point,
          limit
        }
      });

      return {
        el: vNode.el,
        destroy
      };
    },
    getLimitOverlap(limit, overlapMap) {
      //calculate if limit lines are too close to each other
      let limitTop = this.drawAPI.y(limit.point.y);
      const needsVerticalAdjustment = limitTop - CLEARANCE <= 0;
      let needsHorizontalAdjustment = false;
      overlapMap.forEach((value) => {
        let diffTop;
        if (limitTop > value.overlapTop) {
          diffTop = limitTop - value.overlapTop;
        } else {
          diffTop = value.overlapTop - limitTop;
        }

        //need to compare +ves to +ves and -ves to -ves
        if (
          !needsHorizontalAdjustment &&
          Math.abs(diffTop) <= CLEARANCE &&
          value.needsHorizontalAdjustment !== true
        ) {
          needsHorizontalAdjustment = true;
        }
      });

      return {
        needsHorizontalAdjustment,
        needsVerticalAdjustment,
        overlapTop: limitTop
      };
    },
    getLimitProps(limit, overlap) {
      let point = {
        left: 0,
        top: this.drawAPI.y(limit.point.y)
      };
      return {
        limit: Object.assign({}, overlap, limit),
        point
      };
    },
    getLimitLabel(limit, overlap) {
      let point = {
        left: 0,
        top: this.drawAPI.y(limit.point.y)
      };
      const { vNode, destroy } = mount(LimitLabel, {
        props: {
          limit: Object.assign({}, overlap, limit),
          point
        }
      });

      return {
        el: vNode.el,
        destroy
      };
    },
    drawAlarmPoints(alarmSet) {
      this.drawAPI.drawLimitPoints(
        alarmSet.points,
        alarmSet.series.get('color').asRGBAArray(),
        alarmSet.series.get('markerSize')
      );
    },
    drawPoints(chartElement) {
      this.drawAPI.drawPoints(
        chartElement.getBuffer(),
        chartElement.color().asRGBAArray(),
        chartElement.count,
        chartElement.series.get('markerSize'),
        chartElement.series.get('markerShape')
      );
    },
    drawLine(chartElement, disconnected) {
      if (chartElement) {
        this.drawAPI.drawLine(
          chartElement.getBuffer(),
          chartElement.color().asRGBAArray(),
          chartElement.count,
          disconnected
        );
      }
    },
    prepareToDrawAnnotatedPoints(yAxisId) {
      if (this.annotatedPointsBySeries && Object.values(this.annotatedPointsBySeries).length) {
        const uniquePointsToDraw = new Set();

        Object.keys(this.annotatedPointsBySeries).forEach((seriesKeyString) => {
          const seriesModel = this.getSeries(seriesKeyString);
          const matchesYAxis = this.matchByYAxisId(yAxisId, { series: seriesModel });
          if (!matchesYAxis) {
            return;
          }
          // annotation points are all within range (checked in MctPlot with FlatBush), so we don't need to check
          const annotatedPointBuffer = new Float32Array(
            this.annotatedPointsBySeries[seriesKeyString].length * 2
          );
          Object.values(this.annotatedPointsBySeries[seriesKeyString]).forEach(
            (annotatedPoint, index) => {
              const canvasXValue = this.offset[yAxisId].xVal(annotatedPoint.point, seriesModel);
              const canvasYValue = this.offset[yAxisId].yVal(annotatedPoint.point, seriesModel);
              const drawnPointKey = `${canvasXValue}|${canvasYValue}`;
              if (!uniquePointsToDraw.has(drawnPointKey)) {
                annotatedPointBuffer[index * 2] = canvasXValue;
                annotatedPointBuffer[index * 2 + 1] = canvasYValue;
                uniquePointsToDraw.add(drawnPointKey);
              }
            }
          );
          this.drawAnnotatedPoints(seriesModel, annotatedPointBuffer);
        });
      }
    },
    drawAnnotatedPoints(seriesModel, annotatedPointBuffer) {
      if (annotatedPointBuffer && seriesModel) {
        const color = seriesModel.get('color').asRGBAArray();
        // set transparency
        color[3] = 0.15;
        const pointCount = annotatedPointBuffer.length / 2;
        const shape = seriesModel.get('markerShape');

        this.drawAPI.drawPoints(annotatedPointBuffer, color, pointCount, ANNOTATION_SIZE, shape);
      }
    },
    prepareToDrawAnnotationSelections(yAxisId) {
      if (
        this.annotationSelectionsBySeries &&
        Object.keys(this.annotationSelectionsBySeries).length
      ) {
        Object.keys(this.annotationSelectionsBySeries).forEach((seriesKeyString) => {
          const seriesModel = this.getSeries(seriesKeyString);
          const matchesYAxis = this.matchByYAxisId(yAxisId, { series: seriesModel });
          if (matchesYAxis) {
            const annotationSelectionBuffer = new Float32Array(
              this.annotationSelectionsBySeries[seriesKeyString].length * 2
            );
            Object.values(this.annotationSelectionsBySeries[seriesKeyString]).forEach(
              (annotatedSelectedPoint, index) => {
                const canvasXValue = this.offset[yAxisId].xVal(
                  annotatedSelectedPoint.point,
                  seriesModel
                );
                const canvasYValue = this.offset[yAxisId].yVal(
                  annotatedSelectedPoint.point,
                  seriesModel
                );
                annotationSelectionBuffer[index * 2] = canvasXValue;
                annotationSelectionBuffer[index * 2 + 1] = canvasYValue;
              }
            );
            this.drawAnnotationSelections(seriesModel, annotationSelectionBuffer);
          }
        });
      }
    },
    drawAnnotationSelections(seriesModel, annotationSelectionBuffer) {
      const color = [255, 255, 255, 1]; // white
      const pointCount = annotationSelectionBuffer.length / 2;
      const shape = seriesModel.get('markerShape');

      this.drawAPI.drawPoints(annotationSelectionBuffer, color, pointCount, ANNOTATION_SIZE, shape);
    },
    drawHighlights(yAxisId) {
      if (this.highlights && this.highlights.length) {
        const highlights = this.highlights.filter((highlight) => {
          const series = this.getSeries(highlight.seriesKeyString);
          return this.matchByYAxisId.bind(yAxisId, { series });
        });
        highlights.forEach(this.drawHighlight.bind(this, yAxisId), this);
      }
    },
    getSeries(keyStringToFind) {
      const foundSeries = this.seriesModels.find((series) => {
        return series.keyString === keyStringToFind;
      });
      return foundSeries;
    },
    drawHighlight(yAxisId, highlight) {
      const series = this.getSeries(highlight.seriesKeyString);
      const points = new Float32Array([
        this.offset[yAxisId].xVal(highlight.point, series),
        this.offset[yAxisId].yVal(highlight.point, series)
      ]);

      const color = series.get('color').asRGBAArray();
      const pointCount = 1;
      const shape = series.get('markerShape');

      this.drawAPI.drawPoints(points, color, pointCount, HIGHLIGHT_SIZE, shape);
    },
    drawRectangles(yAxisId) {
      if (this.rectangles) {
        this.rectangles.forEach(this.drawRectangle.bind(this, yAxisId), this);
      }
    },
    drawRectangle(yAxisId, rect) {
      if (!rect.start.yAxisIds || !rect.end.yAxisIds) {
        return;
      }

      const startYIndex = rect.start.yAxisIds.findIndex((id) => id === yAxisId);
      const endYIndex = rect.end.yAxisIds.findIndex((id) => id === yAxisId);
      if (rect.start.y[startYIndex] && rect.end.y[endYIndex]) {
        this.drawAPI.drawSquare(
          [this.offset[yAxisId].x(rect.start.x), this.offset[yAxisId].y(rect.start.y[startYIndex])],
          [this.offset[yAxisId].x(rect.end.x), this.offset[yAxisId].y(rect.end.y[endYIndex])],
          rect.color
        );
      }
    }
  }
};
</script>
