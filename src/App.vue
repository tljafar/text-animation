<template>
  <div class="container max-w-7xl px-3 md:px-12 py-24">
    <div class="grid md:grid-cols-2 gap-6">
      <div class="text-lg font-semibold mb-3">Options</div>
      <div class="text-lg font-semibold mb-3">Preview</div>
    </div>
    <div class="grid md:grid-cols-2 gap-6">
      <div class="bg-container p-6 rounded-md md:h-full">
        <div class="flex flex-wrap gap-3">
          <label class="flex items-center mt-3 border px-3 py-2 cursor-pointer rounded-md font-semibold" :class="{ 'btn-selected': formData.isImportFromCsv === false }">
            <input type="radio" name="animation" class="hidden form-radio border-gray-300 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50" @click="() => formData.isImportFromCsv = false">
            <span>Input Text</span>
          </label>
          <label class="flex items-center mt-3 border px-3 py-2 cursor-pointer rounded-md font-semibold" :class="{ 'btn-selected': formData.isImportFromCsv === true }">
            <input type="radio" name="animation" class="hidden form-radio border-gray-300 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50" @click="() => formData.isImportFromCsv = true">
            <span>Import from CSV</span>
          </label>
        </div>
        <div v-if="!formData.isImportFromCsv">
          <label class="block mt-3">
            <span>Input Text:</span>
            <input type="text" class="mt-3 block w-full bg-transparent rounded-md border-gray-300 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50" v-model="formData.input">
          </label>
          <label class="block mt-3">Animation: </label>
          <div class="flex flex-wrap gap-3">
            <label v-for="(item, index) in animations" :key="index" class="flex items-center mt-3 border px-3 py-2 cursor-pointer rounded-md font-semibold" :class="{ 'btn-selected': formData.animation === item }">
              <input type="radio" name="animation" class="hidden form-radio border-gray-300 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50" v-model="formData.animation" :value="item">
              <span>{{ item }}</span>
            </label>
          </div>
          <label class="block mt-3">Output: </label>
          <div class="flex flex-wrap gap-3">
            <label v-for="(item, index) in outputs" :key="index" class="flex items-center mt-3 border px-3 py-2 cursor-pointer rounded-md font-semibold" :class="{ 'btn-selected': formData.output === item }">
              <input type="radio" name="output" class="hidden form-radio border-gray-300 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50" v-model="formData.output" :value="item">
              <span>{{ item }}</span>
            </label>
          </div>
          <div>
            <label class="block mt-3">
              <span class="block">Animation Duration (ms):</span>
              <input type="number" step="50" class="mt-3 rounded-md bg-transparent border-gray-300 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50" v-model="formData.animation_duration">
            </label>
          </div>
          <div>
            <label class="block mt-3">
              <span class="block">Animation Pause (ms):</span>
              <input type="number" step="50" class="mt-3 rounded-md bg-transparent border-gray-300 shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50" v-model="formData.animation_pause">
            </label>
          </div>
        </div>
        <div v-else>
          <label class="block mt-3">
            <span>Import CSV:</span>
            <input type="file" class="form-file mt-3 block w-full rounded-md border shadow-sm focus:border-indigo-300 focus:ring focus:ring-indigo-200 focus:ring-opacity-50" @change="onFileUpload">
          </label>
        </div>
        <input type="button" class="mt-3 px-3 py-2 font-semibold rounded-md text-black bg-teal-600 hover:bg-teal-500 cursor-pointer transition" value="Generate" @click="GenerateAnimation()">
      </div>
      <div class="bg-container rounded-md p-6 md:h-full">
        <div ref="previewElement" class="mt-6" v-html="formData.result"></div>
      </div>
    </div>
  </div>
  <div class="container mt-6 px-3 md:px-12">
    <div v-if="formData.isImportFromCsv" class="mt-6">
      <table class="border-collapse border border-slate-400">
        <thead>
          <tr>
            <th v-for="tableHead in csvRowDataKeys" :key="tableHead" class="border border-slate-300 p-2">
              {{ tableHead }}
            </th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="(row, index) in csvRowData" :key="index">
            <td v-for="tableHead in csvRowDataKeys" :key="tableHead" class="border border-slate-300 p-2">
              {{ row[tableHead] }}
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>


<script setup>
import { ref, watchEffect } from "vue";
import Papa from 'papaparse'
const OUTPUTS = {
  TEXT: 'Text',
  OBJECT: 'Object in a Grid'
}
const animations = ['Bounce', 'Blink', 'Jello', 'Wobble'];
const outputs = ['Text', 'Object in a Grid'];

const ANIMATION = {
  DURATION: 300,
  PAUSE: 200,
  STYLE: {
    BOUNCE: 'Bounce',
    BLINK: 'Blink',
    JELLO: 'Jello',
    WOBBLE: 'Wobble',
  }
}

const formData = ref({
  isImportFromCsv: false,
  input: 'We can think of %2% x %5% as %2% group of %5% 😀',
  animation: animations[0],
  output: outputs[0],
  animation_duration: ANIMATION.DURATION,
  animation_pause: ANIMATION.PAUSE,
  result: ''
});

const previewElement = ref(null);
const csvRowData = ref([]);
const csvRowDataKeys = ref([]);

watchEffect(() => {
  GenerateAnimation();
});

/**
 * 
 * @param {Array} csvData 
 */
async function makeAnimation(csvData) {
  formData.value.result = '';
  await makePromise(async () => {
    const OUTPUTS_OBJECT = OUTPUTS.OBJECT.split(' ')[0].toLowerCase();
    const items = [];
    let itemsHtml = '';
    for (let rowIndex = 0; rowIndex < csvData.length; rowIndex++) {
      const row = csvData[rowIndex];
      let rowOutput = row.type.toLowerCase().includes(OUTPUTS_OBJECT) ? OUTPUTS.OBJECT : OUTPUTS.TEXT;
      const item = {
        input: row.base,
        animation: row.animation ? row.animation : ANIMATION.STYLE.BOUNCE,
        output: rowOutput,
        animation_duration: row.animation_duration ? parseInt(row.animation_duration) : ANIMATION.DURATION,
        animation_pause: row.animation_pause ? parseInt(row.animation_pause) : ANIMATION.PAUSE,
        result: '',
        colors: [row.param_1, row.param_2]
      };
      item.result = getResults(item, rowIndex);
      itemsHtml += item.result
      items.push(item);
    }
    formData.value.result = itemsHtml;
    animateToQueue(items)
  }, 50);
}

async function animateToQueue(items) {
  await makePromise(async () => {
    if (previewElement.value) {
      for (let rowIndex = 0; rowIndex < items.length; rowIndex++) {
        const item = items[rowIndex];
        const animation_pause = item.animation_pause;
        const elements = Array.from(previewElement.value.querySelectorAll(`.row-${rowIndex} .hidden`))
        for (let index = 0; index < elements.length; index++) {
          const element = elements[index];
          if (rowIndex === 0 && index === 0) {
            toAnimate(element, item);
          } else {
            await makePromise(() => {
              toAnimate(element, item);
            }, animation_pause);
          }
        }
      }

    }
  }, 50);
}

async function makePromise(callback, timeout) {
  return await new Promise(resolve => {
    setTimeout(() => {
      callback();
      resolve();
    }, timeout);
  });
}


function getResults(item, rowIndex) {
  const { colors } = item;
  let number_1 = 1;
  let number_2 = 1;
  let type = '';
  let number_1_style = `color: ${colors[0] ? colors[0] : 'inherit'}; animation-duration: ${item.animation_duration}ms;`;
  let number_2_style = `color: ${colors[1] ? colors[1] : 'inherit'}; animation-duration: ${item.animation_duration}ms;`;

  const regex_number = /(%\d+%)[\sx*]+(%\d+%)/gm;
  const regex_type = /\d+\s*%\s+([\w]+)$|\d+\s*%\s+([\W]+)$/gm;

  let str = item.input.trim().replace(/\.[^.]*$/gm, '');
  let m;
  while ((m = regex_type.exec(str)) !== null) {
    if (m.index === regex_number.lastIndex) regex_number.lastIndex++;
    if (m[2]) type = m[2];
  }
  str = str + '.';
  while ((m = regex_number.exec(str)) !== null) {
    if (m.index === regex_number.lastIndex) regex_number.lastIndex++;
    if (m[1]) {
      number_1 = parseInt(m[1].replaceAll("%", ""));
      str = str.replaceAll(m[1], `<span class="hidden" style="${number_1_style}">${number_1}</span>`);
    }
    if (m[2]) {
      number_2 = parseInt(m[2].replaceAll("%", ""));
      str = str.replaceAll(m[2], `<span class="hidden" style="${number_2_style}">${number_2}</span>`);
    }
  }

  if (item.output == OUTPUTS.OBJECT) {
    str = "";
    [...Array(number_1)].forEach(() => {
      [...Array(number_2)].forEach(() => {
        str += `<span class="hidden" style="animation-duration: ${item.animation_duration}ms;">${type}</span>`;
      });
      str += "</br>";
    });
  }

  return `<div class="row-${rowIndex}">${str}</div>`;
}


function toAnimate(element, item) {
  let elementClasses = [
    'animate',
    'animate-' + item.animation.toLowerCase(),
  ];
  element.className = elementClasses.join(' ');
}

function onFileUpload(e) {
  if (e.target.files[0]) {
    const file = e.target.files[0];
    const reader = new FileReader();
    reader.readAsText(file);
    reader.onload = function (e) {
      const jsonObject = Papa.parse(e.target.result, {
        delimiter: ',',
        dynamicTyping: true,
        header: true,
        skipEmptyLines: true
      });
      setCsvRowData(jsonObject.data);
    }
  } else {
    setCsvRowData([]);
  }
}
async function readStaticCSV() {
  const fileString = `base,animation,animation_duration,animation_pause,type,param_1,param_2,output
We can think of %2% x %8% as %2% group of %8% circles.,bounce,1000,300,text,#FF500D,#3B6404,We can think of 1 x 8 as 1 group of 8 circles.
We can think of %1% x %8% as %1% group of %8% circles.,wobble,2000,2000,text,#FF500D,#3B6404,We can think of 1 x 8 as 1 group of 8 circles.
We can think of %1% x %8% as %1% group of %8% circles.,blink,,,text,#FF500D,#3B6404,We can think of 1 x 8 as 1 group of 8 circles.
We can think of %1% x %8% as %1% group of %8% circles.,jello,500,1000,text,#FF500D,#3B6404,We can think of 1 x 8 as 1 group of 8 circles.
We can think of %2% x %5% as %2% groups of %5% 🔵.,bounce,2000,2000,objectsInAGrid,2,5,
`;
  const jsonObject = Papa.parse(fileString, {
    delimiter: ',',
    dynamicTyping: true,
    header: true,
    skipEmptyLines: true
  });
  setCsvRowData(jsonObject.data);
}

function setCsvRowData(csvData) {
  csvRowData.value = csvData;
  csvRowDataKeys.value = csvData ? Object.keys(csvData[0]) : [];
}

function GenerateAnimation() {
  if (formData.value.isImportFromCsv) {
    readStaticCSV();
    makeAnimation(csvRowData.value);
  } else {
    const { input, animation, output, animation_duration, animation_pause } = formData.value;
    makeAnimation([{
      base: input,
      animation: animation,
      type: output,
      animation_duration: animation_duration,
      animation_pause: animation_pause
    }]);
  }
}

</script>