<template>
  <div>
    <h2>Histogram Chart with Zoom and Selection</h2>

    <!-- File Upload -->
    <label for="csvUpload" class="upload-label">Upload CSV File:</label>
    <input type="file" id="csvUpload" @change="handleFileUpload" accept=".csv" />

    <!-- Main Chart -->
    <canvas ref="myChartRef"></canvas>

    <!-- Button to Clear Lines -->
    <button @click="clearLines" :disabled="lines.length === 0" class="clear-button">Clear Lines</button>

    <!-- Show Selected Range -->
    <div v-if="lines.length === 2" class="range-display">
      <p>Selected Range: {{ lines[0].value }} to {{ lines[1].value }}</p>
    </div>

    <!-- Small Chart Displaying Selected Area -->
    <div v-if="smallChartData" class="small-chart-container">
      <h3>Selected Range Chart</h3>
      <canvas ref="smallChartRef"></canvas>
    </div>

    <!-- Carousel Slider Showing Images in the Selected Range -->
    <div v-if="selectedImages.length > 0" class="carousel-container">
      <h3>Images in Selected Range</h3>
      <div class="carousel-slider">
        <div v-for="(image, index) in selectedImages" :key="index" class="carousel-item">
          <img :src="image.src" :alt="'Image ' + index" class="carousel-image" @click="openImage(image.src)" @error="handleImageError(index)" />
          <div class="image-text">{{ image.range }}</div>
        </div>
      </div>
    </div>

    <!-- Image Pop-up -->
    <div v-if="popupImage" class="popup-overlay" @click="closeImage">
      <div class="popup-content" @click.stop>
        <button class="close-button" @click="closeImage">&times;</button>
        <img :src="popupImage" alt="Popup Image" class="popup-image" />
      </div>
    </div>
  </div>
</template>

<script>
import { Chart, BarController, BarElement, Title, Tooltip, Legend, CategoryScale, LinearScale } from 'chart.js';
import zoomPlugin from 'chartjs-plugin-zoom';
import annotationPlugin from 'chartjs-plugin-annotation';
import { readString } from 'react-papaparse';

Chart.register(BarController, BarElement, CategoryScale, LinearScale, Title, Tooltip, Legend, zoomPlugin, annotationPlugin);

export default {
  name: 'HistogramChart',
  data() {
    return {
      specData: [],
      selectedImages: [],
      popupImage: null,
      chartInstance: null,
      showSecondaryChart: false,
      lines: [], // Array to store line annotations
      smallChartData: null, // Data for the small chart
      smallChartInstance: null, // Instance of the small chart
    };
  },
  mounted() {
    this.initializeChart();
  },
  methods: {
    handleFileUpload(event) {
      const file = event.target.files[0];
      if (file) {
        const reader = new FileReader();
        reader.onload = (e) => {
          const csvString = e.target.result;
          this.parseCSV(csvString);
        };
        reader.readAsText(file);
      }
    },
    parseCSV(csvString) {
      readString(csvString, {
        complete: (result) => {
          const data = result.data.map(d => ({
            x0: parseFloat(d.x0),
            x1: parseFloat(d.x1),
            frequency: parseFloat(d.frequency),
            images: d.image ? d.image.split(';') : [],
          }));

          this.specData = data;
          this.updateChart();
        },
        header: true,
      });
    },
    initializeChart() {
      const ctx = this.$refs.myChartRef.getContext('2d');
      this.chartInstance = new Chart(ctx, {
        type: 'bar',
        data: {
          labels: [],
          datasets: [{
            label: 'Frequency',
            backgroundColor: '#42A5F5',
            data: []
          }]
        },
        options: {
          responsive: true,
          plugins: {
            legend: { position: 'top' },
            title: { display: true, text: 'Histogram of Data' },
            zoom: {
              pan: { enabled: true, mode: 'x' },
              zoom: { wheel: { enabled: true }, pinch: { enabled: true }, mode: 'x' }
            },
            annotation: {
              annotations: {}
            }
          },
          scales: {
            x: { beginAtZero: true },
            y: { beginAtZero: true },
          },
          onClick: this.handleChartClick,
        }
      });
    },
    updateChart() {
      if (!this.chartInstance) return;

      this.chartInstance.data.labels = this.specData.map(d => `${d.x0} to ${d.x1}`);
      this.chartInstance.data.datasets[0].data = this.specData.map(d => d.frequency);
      this.chartInstance.update();
    },
    handleChartClick(event) {
      if (this.lines.length >= 2) {
        console.log("Only two lines allowed");
        return; // Prevent adding more than two lines
      }

      const elements = this.chartInstance.getElementsAtEventForMode(event, 'nearest', { intersect: true }, true);

      if (elements.length) {
        const firstElement = elements[0];
        const datasetIndex = firstElement.datasetIndex;
        const index = firstElement.index;

        const xValue = this.chartInstance.data.labels[index];
        const yValue = this.chartInstance.data.datasets[datasetIndex].data[index];

        console.log('Clicked bar:', { xValue, yValue });

        this.addAnnotationLine(`line-${Date.now()}`, xValue);
        this.lines.push({ id: `line-${Date.now()}`, value: xValue, yValue });

        if (this.lines.length === 2) {
          this.showRange();
          this.$nextTick(() => {
            this.createSmallChart();
          });
        }
      } else {
        console.error('No elements found at the click position');
      }
    },
    addAnnotationLine(id, value) {
      this.chartInstance.options.plugins.annotation.annotations[id] = {
        type: 'line',
        scaleID: 'x',
        value: value,
        borderColor: 'red',
        borderWidth: 2,
      };
      this.chartInstance.update();

      console.log('Annotation added:', this.chartInstance.options.plugins.annotation.annotations);
    },
    clearLines() {
      this.chartInstance.options.plugins.annotation.annotations = {};
      this.chartInstance.update();
      this.lines = [];
      this.smallChartData = null;
      this.selectedImages = [];
      this.popupImage = null;
      if (this.smallChartInstance) {
        this.smallChartInstance.destroy();
        this.smallChartInstance = null;
      }
    },
    showRange() {
      if (this.lines.length === 2) {
        console.log(`Selected range: ${this.lines[0].value} to ${this.lines[1].value}`);
      }
    },
    createSmallChart() {
      const minSelection = Math.min(this.lines[0].value.split(' ')[0], this.lines[1].value.split(' ')[0]);
      const maxSelection = Math.max(this.lines[0].value.split(' ')[2], this.lines[1].value.split(' ')[2]);

      console.log(`Filtering data between ${minSelection} and ${maxSelection}`);

      const filteredData = this.specData.filter(d => d.x0 >= minSelection && d.x1 <= maxSelection);

      console.log('Filtered Data:', filteredData);

      if (filteredData.length === 0) {
        console.warn('No data points found within the selected range');
        return;
      }

      this.smallChartData = {
        labels: filteredData.map(d => `${d.x0} to ${d.x1}`),
        datasets: [{
          label: 'Frequency',
          backgroundColor: '#FFA726',
          data: filteredData.map(d => d.frequency)
        }]
      };

      this.selectedImages = filteredData.flatMap(d => d.images.map(image => ({ src: image, range: `${d.x0} to ${d.x1}` })));

      this.$nextTick(() => {
        const canvas = this.$refs.smallChartRef;
        if (canvas) {
          const ctx = canvas.getContext('2d');
          if (this.smallChartInstance) {
            this.smallChartInstance.destroy();
          }
          this.smallChartInstance = new Chart(ctx, {
            type: 'bar',
            data: this.smallChartData,
            options: {
              responsive: true,
              plugins: {
                legend: { position: 'top' },
                title: { display: true, text: 'Selected Range Data' }
              },
              scales: {
                x: { beginAtZero: true },
                y: { beginAtZero: true },
              }
            }
          });
        } else {
          console.error('Small chart canvas not found');
        }
      });
    },
    openImage(imageSrc) {
      this.popupImage = imageSrc;
    },
    closeImage() {
      this.popupImage = null;
    },
    handleImageError(index) {
      this.selectedImages[index].src = 'https://via.placeholder.com/150?text=No+Image';
    },
  },
};
</script>

<style scoped>
.carousel-container {
  margin-top: 30px;
}

.carousel-slider {
  display: flex;
  overflow-x: auto;
  scroll-snap-type: x mandatory;
  padding: 10px;
  border: 1px solid #ccc;
  border-radius: 10px;
  background-color: #f9f9f9;
  min-height: 150px;
}

.carousel-item {
  flex: 0 0 auto;
  width: 150px;
  margin-right: 10px;
  scroll-snap-align: start;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
}

.carousel-image {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
  border-radius: 5px;
  cursor: pointer;
}

.image-text {
  margin-top: 5px;
  font-size: 12px;
  text-align: center;
}

.upload-label {
  display: block;
  margin-bottom: 10px;
  font-weight: bold;
}

.clear-button {
  margin-top: 20px;
  padding: 10px 20px;
  background-color: #f44336;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}

.clear-button:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}

.range-display {
  margin-top: 20px;
  font-weight: bold;
}

.small-chart-container {
  margin-top: 30px;
  width: 100%;
  max-width: 600px;
}

.popup-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.popup-content {
  position: relative;
  max-width: 90%;
  max-height: 90%;
  background-color: white;
  border-radius: 10px;
  padding: 20px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
}

.popup-image {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
  border-radius: 10px;
}

.close-button {
  position: absolute;
  top: -10px;
  right: -10px;
  background-color: #fff;
  border: none;
  font-size: 24px;
  line-height: 24px;
  padding: 5px;
  border-radius: 50%;
  cursor: pointer;
  z-index: 1001;
}
</style>
