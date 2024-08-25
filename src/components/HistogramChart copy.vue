<template>
  <div>
    <h2>Histogram Chart with Zoom and Frequency Selection</h2>

    <!-- File Upload -->
    <label for="csvUpload" class="upload-label">Upload CSV File:</label>
    <input type="file" id="csvUpload" @change="handleFileUpload" accept=".csv" />

    <!-- Dropdown for selecting frequency range -->
    <label for="frequency-range">Select Frequency Range:</label>
    <select id="frequency-range" v-model="selectedFrequencyRange" @change="filterData">
      <option v-for="(range, index) in frequencyRanges" :key="index" :value="range">
        {{ range.label }}
      </option>
    </select>

    <!-- Main Chart -->
    <Bar
      id="my-chart-id"
      :data="filteredChartData"
      :options="chartOptions"
    />

    <!-- Secondary Chart with Carousel Slider -->
    <div v-if="showSecondaryChart" class="carousel-container">
      <h3>Selected Range Details</h3>
      <div class="carousel-slider">
        <div v-if="selectedImages.length === 0">No images available for the selected range.</div>
        <div v-for="(image, index) in selectedImages" :key="index" class="carousel-item">
          <img :src="image" :alt="'Image ' + index" class="carousel-image" @error="handleImageError(index)" />
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { Bar } from 'vue-chartjs';
import { Chart, Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale } from 'chart.js';
import zoomPlugin from 'chartjs-plugin-zoom';
import { readString } from 'react-papaparse';

// Register the zoom plugin with Chart.js
Chart.register(zoomPlugin, Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale);

export default {
  name: 'HistogramChart',
  components: { Bar },
  data() {
    return {
      specData: [],
      frequencyRanges: [
        { label: 'All', min: -Infinity, max: Infinity },
        { label: '0 - 100', min: 0, max: 100 },
        { label: '100 - 200', min: 100, max: 200 },
        { label: '200 - 300', min: 200, max: 300 },
        { label: '300 - 400', min: 300, max: 400 },
        { label: '400 - 500', min: 400, max: 500 },
        { label: '500+', min: 500, max: Infinity },
      ],
      selectedFrequencyRange: { label: 'All', min: -Infinity, max: Infinity },
      filteredChartData: {
        labels: [],
        datasets: []
      },
      selectedImages: [],
      chartOptions: {
        responsive: true,
        plugins: {
          legend: {
            position: 'top',
          },
          title: {
            display: true,
            text: 'Histogram of Frequency Data',
          },
          zoom: {
            pan: {
              enabled: true,
              mode: 'x', // Allow panning in the x direction
            },
            zoom: {
              wheel: {
                enabled: true, // Enable zooming with mouse wheel
              },
              pinch: {
                enabled: true, // Enable zooming by pinching on touch devices
              },
              mode: 'x', // Only zoom in the x direction
            },
          },
        },
        scales: {
          x: {
            beginAtZero: true,
          },
          y: {
            beginAtZero: true,
          },
        },
      },
      showSecondaryChart: false,
    };
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
            x0: parseFloat(d.x0),       // Convert x0 to a float
            x1: parseFloat(d.x1),       // Convert x1 to a float
            frequency: parseFloat(d.frequency),  // Convert frequency to a float
            images: d.image ? d.image.split(';') : []  // Assuming images are semicolon-separated
          }));

          // Debug: Log the parsed data
          console.log('Parsed CSV data:', data);

          // Assign the parsed data to specData
          this.specData = data;

          // Initialize the chart with full data
          this.filterData();
        },
        header: true, // Ensures the first row is treated as headers
      });
    },
    filterData() {
      const filteredValues = this.specData.filter(d =>
        d.frequency >= this.selectedFrequencyRange.min &&
        d.frequency <= this.selectedFrequencyRange.max
      );

      this.filteredChartData = {
        labels: filteredValues.map(d => `${d.x0} to ${d.x1}`),
        datasets: [
          {
            label: 'Frequency',
            backgroundColor: '#42A5F5',
            data: filteredValues.map(d => d.frequency),
          },
        ],
      };

      // Prepare the selected images based on the selected range
      this.prepareSelectedImages(filteredValues);
      this.showSecondaryChart = this.selectedImages.length > 0;
      console.log('showSecondaryChart:', this.showSecondaryChart); // Debug log
    },
    prepareSelectedImages(filteredValues) {
      this.selectedImages = [];
      filteredValues.forEach(value => {
        this.selectedImages.push(...value.images);
      });
      console.log('Selected Images:', this.selectedImages); // Debug log
    },
    handleImageError(index) {
      // Replace the broken image link with a placeholder or handle it accordingly
      this.selectedImages[index] = 'https://via.placeholder.com/300?text=No+Image';
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
  min-height: 200px; /* Ensure the carousel has a minimum height */
}

.carousel-item {
  flex: 0 0 auto;
  width: 300px;
  height: 200px;
  margin-right: 20px;
  scroll-snap-align: start;
  display: flex;
  align-items: center;
  justify-content: center;
}

.carousel-image {
  max-width: 100%;
  max-height: 100%;
  object-fit: contain;
  border-radius: 5px;
}

.upload-label {
  display: block;
  margin-bottom: 10px;
  font-weight: bold;
}
</style>
