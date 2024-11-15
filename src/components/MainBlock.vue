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
    handleSubmit(value)
  }
}

handleSubmit('UUWW')

async function handleSubmit(stationLocator) {
  try {
    // Выполняем GET-запрос к API
    const response = await axios.get(
      `https://api.vatrus.info/v1/metar/${stationLocator}/extended`,
    )

    // Получаем данные из ответа
    // const metarData = response.data
    // Логируем полученные данные
    // console.log('Получены данные METAR:', metarData)

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
        if (item[0] != null) {
          let rwyDesignator = ''
          let rwyDesignatorText = ''
          if (item[1] != null) {
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
                if (item[2] != null) {
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
          if (item[4] != null) {
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

          if (item[5] != null) {
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
          if (item[6] != null) {
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
            let param = parseInt(item[6], 10)
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

          if (item[7] != null) {
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

          if (item != null)
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

    const patternVerticalVisibility = /VV(\d{3}|\/\/\/)/g
    let matchesVV = []
    let matchVV
    while ((matchVV = patternVerticalVisibility.exec(raw)) !== null) {
      matchesVV.push(matchVV)
    }
    if (matchesVV) {
      matchesVV.forEach(item => {
        let VVText = ''
        if (item[1] == '///') {
          VVText += 'The vertical visibility has not been measured'
        } else {
          VVText +=
            'Vertical visibility of ' + parseInt(item[1], 10) * 100 + 'ft'
        }
        addNewString(replacements, item[0], VVText, 'VerticalVisibility', 'red')
      })
    }

    /*Atmospheric pressure */

    const patternAtmosphericPressure = /Q(\d{4})/g
    let matchesAP = []
    let matchAP
    while ((matchAP = patternAtmosphericPressure.exec(raw)) !== null) {
      matchesAP.push(matchAP)
    }
    if (matchesAP) {
      matchesAP.forEach(item => {
        let APText =
          'The sea level pressure or QNH at the aerodrome is ' +
          item[1] +
          ' hectopascal'

        addNewString(
          replacements,
          item[0],
          APText,
          'Atmospheric Pressure',
          'red',
        )
      })
    }

    /**Air temperature and dew point */
    const patternTempAndDewPoint = /\b((M)?(\d{2}))\/((M)?(\d{2}))\b/g
    let matchesTDP = []
    let matchTDP
    while ((matchTDP = patternTempAndDewPoint.exec(raw)) !== null) {
      matchesTDP.push(matchTDP)
    }
    if (matchesTDP) {
      matchesTDP.forEach(item => {
        let tempText = ''
        let dewPointText = ''

        if (item[3] !== null) {
          tempText += (item[2] == 'M' ? '-' : '') + parseInt(item[3], 10)
        }
        if (item[6] !== null) {
          dewPointText += (item[5] == 'M' ? '-' : '') + parseInt(item[6], 10)
        }
        console.log(item)
        addNewString(
          replacements,
          item[0],
          'Air temperature is ' +
            tempText +
            '°C, dew point temperature is ' +
            dewPointText +
            '°C',
          'Air temperature and dew point',
          'red',
        )
      })
    }

    // QNH

    /**Clouds */

    const patternClouds =
      /(FEW|SCT|BKN|OVC|\/\/\/)(\d{3}|\/\/\/)(CB|TCU|\/\/\/)?/g

    // let input1 =
    //   ' BKN013 ///015 SCT/// BKN///CB OVC/// ///015 ////// //////CB BKN025///'
    let matchesClouds = []
    let matchClouds
    while ((matchClouds = patternClouds.exec(raw)) !== null) {
      matchesClouds.push(matchClouds)
    }
    if (matchesClouds) {
      matchesClouds.forEach(item => {
        let cloudsText = ''
        let cloudAmount = ''
        let cloudsHeight = ''
        let cloudType = ''

        cloudAmount += ''
        if (item[1] !== null) {
          switch (item[1]) {
            case 'FEW':
              cloudAmount +=
                'Few clouds layer (1/8th to 2/8th of sky coverage (1-2 octas))'
              break
            case 'SCT':
              cloudAmount +=
                'Scattered clouds layer (3/8th to 4/8th of sky coverage (3-4 octas))'
              break
            case 'BKN':
              cloudAmount +=
                'Broken clouds layer (5/8th to 7/8th of sky coverage (5-6-7 octas))'
              break
            case 'OVC':
              cloudAmount +=
                'Overcast clouds layer (8/8th of sky coverage (8 octas))'
              break
            case '///':
              cloudAmount +=
                'The cloud amount cannot be identified by the automatic observing system; '
              break
          }
        }
        //todo ??
        if (item[2] != null) {
          cloudsHeight = ' at ' + parseInt(item[2], 10) * 100 + 'ft'
        } else {
          cloudsHeight = ' the layer is below station level'
        }
        if (item[3] != null) {
          cloudType += ''

          switch (item[3]) {
            case 'TCU':
              cloudType += ' with presence of towering cumulus (TCU)'
              break
            case 'CB':
              cloudType += ' with presence of cumulonimbus (CB)'
              break
            case '///':
              cloudType +=
                'The cloud type cannot be identified by the automatic observing system; '
          }
        }
        // console.log(item)
        // console.log(cloudsText + cloudAmount + cloudsHeight + cloudType)
        addNewString(
          replacements,
          item[0],
          cloudsText + cloudAmount + cloudsHeight + cloudType,
          'Clouds',
          'red',
        )
      })
    }

    /** Wind */
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
          return +parseInt(speed, 10) + getWindUnits(units) + '; '
        }

        if (item[1] == 'VRB') {
          speedText += 'variable direction; wind speed '
          speedText += getWindSpeed(item[4], item[6])
          speedText +=
            '<p><b>VRB is used:</b></p><ul><li>If the wind direction changes in the range from 60&deg; to 180&deg;, and the wind speed is less than 1.5 m/s (3 kt)</li><li>If the wind direction changes by 180 degrees or more, such as during a thunderstorm passing over an airfield</li></ul>'
        } else if (item[3] == 'P' || item[3] == 'ABV') {
          if (item[1] != null) {
            directionText +=
              'surface wind direction ' + parseInt(item[1], 10) + '°; '
          }
          speedText += 'wind speed of 50 m/s (100KT) or more'
        } else {
          if (item[1] != null) {
            directionText +=
              'surface wind direction ' + parseInt(item[1], 10) + '°; '
          }
          speedText += ' wind speed '
          speedText += getWindSpeed(item[4], item[6])

          if (item[5] != null) {
            gustText +=
              ' gusting to a maximum of ' +
              parseInt(item[5], 10) +
              getWindUnits(item[6]) +
              '; '
          }

          if (item[1] != null && item[8] != null && item[9] != null) {
            windDirectionVariableText +=
              ' wind direction is variable around ' +
              parseInt(item[1], 10) +
              '° between ' +
              parseInt(item[8], 10) +
              '° and ' +
              parseInt(item[9], 10) +
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
      })
    }

    const modifiedText = replaceAllOccurrencesInString(raw)

    parsedString.value = modifiedText
  }
})
</script>

<template>
  <div>
    <!-- <input v-model="airportValue" type="text" /> -->
    <div class="mtr-field">
      <label class="mtr-label">Enter airport ICAO index</label>

      <input
        :value="state.icao"
        @input="handleInputChange"
        maxlength="4"
        type="text"
        class="mtr-input"
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
    <h2>Weather for {{ state.airport }}</h2>

    <div class="mtr-raw" v-html="parsedString"></div>
  </div>
  <div v-else></div>
</template>
