<script setup>
import { ref, toRefs, reactive, watchEffect } from 'vue'

import axios from 'axios'

import data from '/src/assets/codes.json'

const state = reactive({
  isLoaded: false,
  icao: '',
  rawText: '',
  airport: '',
  city: '',
  country: '',
})
const { isLoaded } = toRefs(state)
const parsedString = ref('')
const airportValue = ref('UUWW')

function handleInputChange(event) {
  const value = event.target.value
  if (value.length === 4) {
    // Вызываем нужную функцию, когда введено 4 символа
    handleSubmit(value)
  }
}
// function handleSubmit() {
//   console.log('Значение ввода:', airportValue.value)
// }
async function handleSubmit(stationLocator) {
  try {
    // Выполняем GET-запрос к API
    const response = await axios.get(
      `https://api.vatrus.info/v1/metar/${stationLocator}/extended`,
    )

    // Получаем данные из ответа
    const metarData = response.data

    // Логируем полученные данные
    console.log('Получены данные METAR:', metarData)

    if (response.status === 200 && response.data) {
      state.isLoaded = true
      state.icao = response.data.icao
      state.rawText = response.data.raw_text
      state.airport = response.data.airport
    }
    // Здесь можно обработать данные дальше, например, вывести их на экран
  } catch (error) {
    if (axios.isAxiosError(error)) {
      // Логируем ошибку, если она произошла
      console.error('Ошибка при получении данных:', error.message)
    }
  }
}

watchEffect(() => {
  let raw = state.rawText
  // let raw =
  //   'UUEE 070430Z 31005MPS 270V340 3000 2000NW 1000N  OVC009 03/02 Q1020 R24L/290051 R24C/290051 NOSIG 9999'

  if (raw) {
    //https://wiki.ivao.aero/en/home/training/documentation/METAR_explanation
    //http://www.moratech.com/aviation/metaf-abbrev.html

    const replacements = data

    function replaceAllOccurrencesInString(updatedText) {
      replacements.forEach(({ keyword, text, category, color }) => {
        const regex = new RegExp(`\\b${keyword}\\b`, 'g')

        if (regex.test(updatedText)) {
          updatedText = updatedText.replace(
            regex,
            `<span class="code code-color-${color ? color : 'default'}"><span class="code-key">${keyword}</span><span class="code-tooltip"><span class="code-category">${category}</span><span class="code-text">${text}</span></span></span>`,
          )
        }
      })

      return updatedText
    }
    function addNewString(replacements, keyword, text, category, color) {
      replacements.push({
        keyword: keyword,
        text: text,
        category: category,
        color: color,
      })
    }
    /**icao index */
    const icaoIndex = raw.match(/^[A-Z]{4}\b/)[0]

    if (icaoIndex) {
      addNewString(
        replacements,
        icaoIndex,
        icaoIndex + ' stands for ' + state.airport,
        'ICAO Station ID',
      )
    }
    /** time of the observation*/
    const dateTimeUTC = raw.match(/\d{6}Z/)[0]

    const observationDay = dateTimeUTC.substring(0, 2)
    const observationTimeHours = dateTimeUTC.substring(2, 4)
    const observationTimeMinutes = dateTimeUTC.substring(4, 6)

    if (dateTimeUTC) {
      addNewString(
        replacements,
        dateTimeUTC,
        'the date of the weather observation is number ' +
          observationDay +
          ' of the current month, at ' +
          observationTimeHours +
          ':' +
          observationTimeMinutes,
        'Time of the observation',
      )
    }
    // icao #E31E25 red
    // date #FFCD02 yellow
    // wind #EF7F1A orange
    // visib #009847 green
    // weather #AE4B84  purple
    // clouds #393285 violet
    // davlen #B0CA1F salad
    // trend #72635F brown
    // rmk #FFCD02 yellow
    // obsc #009847 green

    /** visibility */
    const visibilityMainPattern =
      /\b\d{4}\b(?:\s\d{4}(NW|NE|SW|SE|N|S|E|W))?(?:\s\d{4}(NW|NE|SW|SE|N|S|E|W))?/g

    const visibility4digitPattern = /\b\d{4}\b/
    const visibilityLowestPattern = /\b\d{4}(NW|NE|SW|SE|N|S|E|W)/

    const visibility = raw.match(visibilityMainPattern)
    if (visibility) {
      visibility.forEach(item => {
        const visibilityParameters = item.split(' ')
        let text = ''
        if (visibilityParameters.length > 1) {
          visibilityParameters.forEach(el => {
            if (el.match(visibility4digitPattern)) {
              text +=
                el +
                ' m <small>is the greatest distance at which a black object of suitable dimensions, situated near the ground, can be seen and recognized when observed against a bright background; and the greatest distance at which lights in the vicinity of 1 000 candelas can be seen and identified against an unlit background.</small>'
            }
            if (el.match(visibilityLowestPattern)) {
              text +=
                '<br/>' +
                el.match(visibilityLowestPattern)[0].substring(0, 4) +
                'm at ' +
                el.match(visibilityLowestPattern)[1]
            }
          })
          addNewString(replacements, item, text, 'Visibility', 'green')
        } else {
          if (visibilityParameters[0] == '9999') {
            text =
              '<small>When visibility is 10 km and above and the conditions for the use of CAVOK <b>do not apply</b>, visibility shall be indicated as 9999.</small>'
          } else if (visibilityParameters[0] == '0000') {
            text =
              '<small>When visibility is below 50m, visibility shall be indicated as 0000.</small>'
          }
          addNewString(replacements, item, text, 'Visibility', 'green')
        }
      })
    }

    const rvrPattern = /\bR(\d{2})([LRC]?)\/([PM]?)(\d{4})([UDN]?)\b/g
    const rvr = raw.match(rvrPattern)

    if (rvr) {
      rvr.forEach(item => {
        let text = ''
        addNewString(
          replacements,
          item,
          text,
          'RVR - Runway visual range / Visibility',
          'red',
        )
      })
    }

    // R24L/451293 R14/////// R88/////// R/SNOCLO R14///99//  R14/CLRD// R88/CLRD//

    const RSGPattern =
      /\bR(\d{2})([LRC]?)\/(([0-9]|\/)(1|2|5|9|\/)(\d{2}|\/\/)|CLRD)(\d{2}|\/\/)\b/g
    // const RSGs = raw.match(RSGPattern)

    let matchesRSG = []
    let matchRSG
    while ((matchRSG = RSGPattern.exec(raw)) !== null) {
      matchesRSG.push(matchRSG)
    }

    if (matchesRSG) {
      matchesRSG.forEach(item => {
        if (item[0]) {
          let rwyDesignator = ''
          let rwyDesignatorText = ''
          if (item[1]) {
            rwyDesignator = item[1]
            switch (rwyDesignator) {
              case 99:
                rwyDesignatorText +=
                  'repetition of the last message as no new information received'
                break
              case 88:
                rwyDesignatorText += 'All runways'
                break
              default:
                rwyDesignatorText += 'Runway ' + rwyDesignator
                if (item[2]) {
                  const sideMap = {
                    L: 'L (left)',
                    R: 'R (right)',
                    C: 'C (center)',
                  }
                  rwyDesignatorText += sideMap[item[2]]
                }
            }
          }
          // console.log(item)
          let clrdText = ''
          if (item[3] == 'CLRD') {
            clrdText += ' cleared;'
          }
          let rwyDepositsText = ''
          if (item[4]) {
            rwyDepositsText += '<br/>Runway deposits: '
            const runwayDepositsMap = {
              0: 'Clear and dry',
              1: 'Damp',
              2: 'Wet or water patches',
              3: 'Rime or frost covered (depth normally less than 1 mm)',
              4: 'Dry snow',
              5: 'Wet snow',
              6: 'Slush',
              7: 'Ice',
              8: 'Compacted or rolled snow',
              9: 'Frozen ruts or ridges',
            }
            if (item[4] == '/') {
              rwyDepositsText +=
                'Type of deposit not reported (e.g., due to runway clearance in progress)'
            } else {
              rwyDepositsText +=
                runwayDepositsMap[item[4]] + ' (' + item[4] + ') '
            }
          }

          let extentOfContaminationText = ''

          if (item[5]) {
            extentOfContaminationText += '<br/>Extent of runway contamination: '
            const extentOfContaminationMap = {
              1: '10% or less',
              2: '11% to 25%',
              5: '26% to 50%',
              9: '51% to 100%',
              '/': 'Not reported (e.g., due to runway clearance in progress)',
            }
            let param = item[5]
            const extentOfContamination = Object.keys(
              extentOfContaminationMap,
            ).find(key => key === param)

            if (extentOfContamination) {
              extentOfContaminationText += `${extentOfContaminationMap[param]} (${param})`
            }
          }

          let depthOfDepositText = ''
          if (item[6]) {
            depthOfDepositText += '<br/>Depth of Deposit: '
            const depthOfDepositMap = {
              92: '10cm',
              93: '15cm',
              94: '20cm',
              95: '25cm',
              96: '30cm',
              97: '35cm',
              98: '40cm or more',
            }
            let param = Number(item[6])
            if (param >= 1 && param <= 90) {
              depthOfDepositText += '' + param + 'mm (' + param + ')'
            } else if (param == '00') {
              depthOfDepositText += 'less than 1mm (00)'
            } else if (param == '99') {
              depthOfDepositText +=
                'rwy(s) non-operational due to snow, slush, ice, large drifts or runway clearance, but depth not reported.'
            } else {
              const depthOfDeposit = Object.keys(depthOfDepositMap).find(
                key => key === param,
              )
              if (depthOfDeposit) {
                depthOfDepositText = `${param} = ${depthOfDepositMap[param]}`
              }
            }
          }
          let frictionCoefficientText = ''

          if (item[7]) {
            frictionCoefficientText += '<br/>Braking Action: '
            const frictionCoefficient = item[7]
            if (frictionCoefficient == '//') {
              frictionCoefficientText +=
                'not reported (e.g. runway not operational; aerodrome closed; etc.)'
            }
            if (frictionCoefficient >= 91) {
              switch (frictionCoefficient) {
                case 91:
                  frictionCoefficientText += 'Poor'
                  break
                case 92:
                  frictionCoefficientText += 'Medium/Poor'
                  break
                case 93:
                  frictionCoefficientText += 'Medium'
                  break
                case 94:
                  frictionCoefficientText += 'Medium/Good'
                  break
                case 95:
                  frictionCoefficientText += 'Good'
                  break
                case 99:
                  frictionCoefficientText +=
                    'Figures unreliable (e.g. if equipment has been used which does not measure satisfactorily in slush or snow.)'
                  break
              }
            } else {
              frictionCoefficientText += ' ' + frictionCoefficient / 100
            }
          }

          if (item)
            addNewString(
              replacements,
              item[0],
              rwyDesignatorText +
                clrdText +
                rwyDepositsText +
                extentOfContaminationText +
                depthOfDepositText +
                frictionCoefficientText,
              'RSG - Runway State Group',
              'red',
            )
        }
      })
    }

    /** wind */
    const patternWithGustings =
      /\b(\d{3}|(VRB))((?:P|ABV)?)(\d{2})(?:G(\d{2}))?(MPS|KPH|KT)\b(\s(\d{3})V(\d{3}))?/g

    let input =
      '31005G10MPS 10005G10MPS 00000MPS 32020MPS 31005MPS 280V350 VRB15MPS VRB01MPS 140P99KT 240P49MPS'

    let matchesWind = []
    let matchWind
    while ((matchWind = patternWithGustings.exec(raw)) !== null) {
      matchesWind.push(matchWind)
    }

    if (matchesWind) {
      matchesWind.forEach(item => {
        let directionText = ''
        let speedText = ''
        let gustText = ''
        let windDirectionVariableText = ''
        let noteText = ''

        function getWindUnits(param) {
          let units = ''
          switch (param) {
            case 'MPS':
              units += '&nbsp;meter(s) per&nbsp;second'
              break
            case 'KPH':
              units += ' kilometres per&nbsp;hour'
              break
            case 'KT':
              units += ' knots'
              break
          }

          return units
        }
        function getWindSpeed(speed, units) {
          return +Number(speed) + getWindUnits(units) + '; '
        }

        if (item) {
          if (item[1] == 'VRB') {
            speedText += 'variable direction; wind speed '
            speedText += getWindSpeed(item[4], item[6])
            speedText +=
              '<p><b>VRB is used:</b></p><ul><li>If the wind direction changes in the range from 60&deg; to 180&deg;, and the wind speed is less than 1.5 m/s (3 kt)</li><li>If the wind direction changes by 180 degrees or more, such as during a thunderstorm passing over an airfield</li></ul>'
          } else if (item[3] == 'P' || item[3] == 'ABV') {
            if (item[1]) {
              directionText +=
                'surface wind direction ' + Number(item[1]) + '°; '
            }
            speedText += 'wind speed of 50 m/s (100KT) or more'
          } else {
            if (item[1]) {
              directionText +=
                'surface wind direction ' + Number(item[1]) + '°; '
            }
            speedText += ' wind speed '
            speedText += getWindSpeed(item[4], item[6])

            if (item[5]) {
              gustText +=
                ' gusting to a maximum of ' +
                Number(item[5]) +
                getWindUnits(item[6]) +
                '; '
            }

            if (item[1] && item[8] && item[9]) {
              windDirectionVariableText +=
                ' wind direction is variable around ' +
                Number(item[1]) +
                '° between ' +
                Number(item[8]) +
                '° and ' +
                Number(item[9]) +
                '°.'
            }
            if (item[7] == null || typeof item[7] === 'undefined') {
            } else {
              noteText +=
                '<br/><br/>If, during the ten minutes preceding the observation, the wind direction changes within the range from 60° to 180°, and the wind speed is 1.5 m/s (3 kt) or more, then the report shall additionally include two extreme values ​​of direction within which the change in surface wind direction was observed, and shall be indicated in clockwise order.'
            }
          }

          addNewString(
            replacements,
            item[0],
            directionText +
              speedText +
              gustText +
              windDirectionVariableText +
              noteText,
            'Wind',
            'red',
          )
        }
      })
    }

    const modifiedText = replaceAllOccurrencesInString(raw)

    parsedString.value = modifiedText
  }
})

// axios
//   .get('https://api.vatrus.info/v1/metar/' + airportValue.value + '/extended')
//   .then(response => {
//     if (response.status === 200 && response.data) {
//       state.isLoaded = true
//       state.icao = response.data.icao
//       state.rawText = response.data.raw_text
//       state.airport = response.data.airport
//     }
//   })
//   .catch(error => {
//     console.error('Error fetching data:', error)
//   })
</script>

<template>
  <div>
    <!-- <input v-model="airportValue" type="text" /> -->
    <label class="mtr-label">Enter airport ICAO index</label>
    <div>
      <input
        :value="state.icao"
        @input="handleInputChange"
        maxlength="4"
        type="text"
        class="mtr-field"
        placeholder=""
      />
    </div>
    <!-- <button @click="handleSubmit">Show</button> -->
  </div>
  <!-- <div class="mtr-suggestions">
    <div class="mtr-suggestions-item" data-saggestion="UUWW">UUWW</div>
    <div class="mtr-suggestions-item" data-saggestion="UUEE">UUEE</div>
    <div class="mtr-suggestions-item" data-saggestion="KSAN">KSAN</div>
  </div> -->

  <div v-if="isLoaded">
    <h2>
      {{ state.airport }}
    </h2>

    <div class="mtr-raw" v-html="parsedString"></div>
  </div>
  <div v-else>Loading...</div>
</template>
