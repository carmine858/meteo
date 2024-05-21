<template>
  <div>
    <h1>Chart from Excel</h1>
    <input type="file" @change="handleFileUpload" />
    <div v-if="dataLoaded">
      <h2>Temperature Table</h2>
      <table>
        <thead>
          <tr>
            <th>Comune</th>
            <th v-for="year in temperatureYears" :key="year">{{ year }}</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(item, index) in temperatureData" :key="index">
            <td>{{ item.comune }}</td>
            <td v-for="year in temperatureYears" :key="year">{{ item.temperatures[year] }}</td>
          </tr>
        </tbody>
      </table>

      <h2>Precipitation Table</h2>
      <table>
        <thead>
          <tr>
            <th>Comune</th>
            <th v-for="year in precipitationYears" :key="year">{{ year }}</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(item, index) in precipitationData" :key="index">
            <td>{{ item.comune }}</td>
            <td v-for="year in precipitationYears" :key="year">{{ item.precipitations[year] }}</td>
          </tr>
        </tbody>
      </table>

      <apexchart type="line" height="350" :options="chartOptions" :series="temperatureSeries"></apexchart>
      <apexchart type="line" height="350" :options="chartOptions" :series="precipitationSeries"></apexchart>
    </div>
  </div>
</template>

<script>
import * as XLSX from 'xlsx';
import { ref } from 'vue';
import VueApexCharts from 'vue3-apexcharts';

export default {
  name: 'HomeView',
  components: {
    apexchart: VueApexCharts
  },
  setup() {
    const temperatureData = ref([]);
    const precipitationData = ref([]);
    const temperatureYears = ref([]);
    const precipitationYears = ref([]);
    const dataLoaded = ref(false);
    const chartOptions = ref({
      chart: {
        type: 'line'
      },
      xaxis: {
        type: 'category' // Cambiato da 'datetime' a 'category' perché si tratta di dati categorici (nomi dei comuni)
      }
    });
    const temperatureSeries = ref([]);
    const precipitationSeries = ref([]);

    const handleFileUpload = (event) => {
      const file = event.target.files[0];
      const reader = new FileReader();

      reader.onload = (e) => {
        const data = new Uint8Array(e.target.result);
        const workbook = XLSX.read(data, { type: 'array' });
        const sheetName = workbook.SheetNames[0];
        const worksheet = workbook.Sheets[sheetName];
        const jsonData = XLSX.utils.sheet_to_json(worksheet, { header: 1 });

        convertExcelToJson(jsonData);
      };

      reader.readAsArrayBuffer(file);
    };

    const convertExcelToJson = (data) => {
      const yearColumns = data[3].slice(1); // Anni nella quarta riga
      const splitIndex = yearColumns.findIndex(year => isNaN(year)); // Trova l'indice che divide i dati di temperatura e precipitazione

      temperatureYears.value = yearColumns.slice(0, splitIndex);
      precipitationYears.value = yearColumns.slice(splitIndex);

      const tempData = [];
      const precipData = [];

      let i = 5; // Inizia a leggere i dati dalla sesta riga
      while (i < data.length && data[i][0] !== "") {
        const row = data[i];
        const comune = row[0];

        if (!comune || comune.trim() === '') {
          i++;
          continue; // Salta righe vuote
        }

        const temperatures = {};
        const precipitations = {};

        for (let j = 1; j <= splitIndex; j++) {
          temperatures[yearColumns[j - 1]] = row[j] !== undefined && row[j] !== '' ? row[j] : null;
        }

        for (let j = splitIndex + 1; j < row.length; j++) {
          precipitations[yearColumns[j - splitIndex - 1]] = row[j] !== undefined && row[j] !== '' ? row[j] : null;
        }

        tempData.push({ comune, temperatures });
        precipData.push({ comune, precipitations });

        i++;
      }

      temperatureData.value = tempData.filter(item => Object.values(item.temperatures).some(value => value !== null));
      precipitationData.value = precipData.filter(item => Object.values(item.precipitations).some(value => value !== null));
      dataLoaded.value = true;

      generateSeries();
    };

    const generateSeries = () => {
      temperatureSeries.value = temperatureYears.value.map(year => {
        return {
          name: `Temperature ${year}`,
          data: temperatureData.value
            .filter(item => item.temperatures[year] !== null)
            .map(item => {
              return {
                x: item.comune,
                y: item.temperatures[year]
              };
            })
        };
      });

      precipitationSeries.value = precipitationYears.value.map(year => {
        return {
          name: `Precipitation ${year}`,
          data: precipitationData.value
            .filter(item => item.precipitations[year] !== null)
            .map(item => {
              return {
                x: item.comune,
                y: item.precipitations[year]
              };
            })
        };
      });
    };

    return {
      handleFileUpload,
      temperatureData,
      precipitationData,
      temperatureYears,
      precipitationYears,
      dataLoaded,
      chartOptions,
      temperatureSeries,
      precipitationSeries
    };
  }
};
</script>

<style>
table {
  width: 100%;
  border-collapse: collapse;
}

th,
td {
  border: 1px solid #ddd;
  padding: 8px;
}

th {
  background-color: #f2f2f2;
}
</style>




