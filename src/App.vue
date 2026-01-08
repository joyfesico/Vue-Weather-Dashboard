<script setup>
// Vue 3 Composition API Fundamentals:

// 1. Reactive Data (equivalent to data() in Options API)
import { ref } from 'vue'

const city = ref('') // Reactive data for search input
const weather = ref(null) // Reactive data for weather information
const loading = ref(false) // Reactive data for loading state
const error = ref('') // Reactive data for error messages
const showAll = ref(false) // Reactive data for show all cities toggle
const unit = ref('metric') // Reactive data for temperature unit ('metric' or 'imperial')

const API_KEY = '40ee243bf6c567584aa11196f1f5cff4' // Your OpenWeatherMap API key

// Alternative: Using Open-Meteo (free, no API key required)
// Uncomment the lines below and comment out the OpenWeatherMap code above to use this instead

// const USE_OPEN_METEO = true // Set to true to use Open-Meteo instead of OpenWeatherMap

// 2. Methods (functions that can be called from template)
const fetchWeather = async (searchCity = null) => {
  // Handle case where event object might be passed
  if (searchCity && typeof searchCity === 'object' && searchCity.target) {
    searchCity = null; // Reset to use city.value
  }
  const cityToSearch = searchCity || city.value.trim()
  if (!cityToSearch) return

  loading.value = true
  error.value = ''
  weather.value = null

  try {
    let response, data

    // Try OpenWeatherMap first
    try {
      response = await fetch(
        `https://api.openweathermap.org/data/2.5/weather?q=${cityToSearch}&appid=${API_KEY}&units=${unit.value}`
      )

      if (response.ok) {
        data = await response.json()
        // Transform OpenWeatherMap data to match our expected format
        weather.value = {
          name: data.name,
          sys: { country: data.sys.country },
          weather: [{ icon: data.weather[0].icon, description: data.weather[0].description }],
          main: {
            temp: data.main.temp,
            feels_like: data.main.feels_like,
            humidity: data.main.humidity,
            pressure: data.main.pressure
          },
          wind: { speed: data.wind.speed }
        }
        return
      }
    } catch (owmError) {
      console.log('OpenWeatherMap failed, trying Open-Meteo...')
    }

    // Fallback to Open-Meteo (free, no API key)
    // First get coordinates for the city
    const geoResponse = await fetch(
      `https://geocoding-api.open-meteo.com/v1/search?name=${encodeURIComponent(cityToSearch)}&count=1&language=en&format=json`
    )

    if (!geoResponse.ok) {
      throw new Error('Unable to find city coordinates')
    }

    const geoData = await geoResponse.json()

    if (!geoData.results || geoData.results.length === 0) {
      throw new Error(`City "${cityToSearch}" not found`)
    }

    const { latitude, longitude, name, country } = geoData.results[0]

    // Get weather data using coordinates
    const weatherResponse = await fetch(
      `https://api.open-meteo.com/v1/forecast?latitude=${latitude}&longitude=${longitude}&current=temperature_2m,relative_humidity_2m,apparent_temperature,pressure_msl,wind_speed_10m,weather_code&timezone=auto`
    )

    if (!weatherResponse.ok) {
      throw new Error('Weather service temporarily unavailable')
    }

    const weatherData = await weatherResponse.json()

    // Transform Open-Meteo data to match our expected format
    const weatherCode = weatherData.current.weather_code
    const weatherDescriptions = {
      0: 'Clear sky',
      1: 'Mainly clear',
      2: 'Partly cloudy',
      3: 'Overcast',
      45: 'Fog',
      48: 'Depositing rime fog',
      51: 'Light drizzle',
      53: 'Moderate drizzle',
      55: 'Dense drizzle',
      61: 'Slight rain',
      63: 'Moderate rain',
      65: 'Heavy rain',
      71: 'Slight snow fall',
      73: 'Moderate snow fall',
      75: 'Heavy snow fall',
      95: 'Thunderstorm',
      96: 'Thunderstorm with slight hail',
      99: 'Thunderstorm with heavy hail'
    }

    const iconMap = {
      0: '01d', 1: '02d', 2: '03d', 3: '04d',
      45: '50d', 48: '50d',
      51: '09d', 53: '09d', 55: '09d',
      61: '10d', 63: '10d', 65: '10d',
      71: '13d', 73: '13d', 75: '13d',
      95: '11d', 96: '11d', 99: '11d'
    }

    weather.value = {
      name: name,
      sys: { country: country },
      weather: [{ icon: iconMap[weatherCode] || '01d', description: weatherDescriptions[weatherCode] || 'Unknown' }],
      main: {
        temp: weatherData.current.temperature_2m,
        feels_like: weatherData.current.apparent_temperature,
        humidity: weatherData.current.relative_humidity_2m,
        pressure: Math.round(weatherData.current.pressure_msl)
      },
      wind: { speed: weatherData.current.wind_speed_10m }
    }

  } catch (err) {
    if (err.message.includes('fetch')) {
      error.value = 'Network error. Please check your internet connection.'
    } else {
      error.value = err.message
    }
  } finally {
    loading.value = false
  }
}

// Additional methods
const toggleUnit = () => {
  unit.value = unit.value === 'metric' ? 'imperial' : 'metric'
  if (weather.value) {
    fetchWeather(weather.value.name)
  }
}

const toggleShowAll = () => {
  showAll.value = !showAll.value
}

// 3. Lifecycle Hook (equivalent to mounted() in Options API)
import { onMounted } from 'vue'
onMounted(() => {
  // Component is ready - could fetch default weather here if desired
  console.log('Weather Dashboard mounted - ready for user input!')
})
</script>

<template>
  <div class="weather-dashboard">
    <!-- Top Search Bar -->
    <div class="search-container">
      <div class="search-bar">
        <svg class="search-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
          <circle cx="11" cy="11" r="8"></circle>
          <path d="m21 21-4.35-4.35"></path>
        </svg>
        <input
          v-model="city"
          type="text"
          placeholder="Search"
          @keyup.enter="fetchWeather()"
        />
      </div>
    </div>

    <!-- Main Content Grid -->
    <div class="main-content">
      <!-- Left Column - Main Weather -->
      <div class="left-column">
        <!-- Main Weather Card -->
        <div class="main-weather-card">
          <div class="date-section">
            <div class="day">Monday</div>
            <div class="date">07 Jan, 2026</div>
          </div>

          <div class="weather-main">
            <div class="weather-info">
              <div class="temperature">
                <span class="temp-number">{{ weather ? Math.round(weather.main.temp) : '26' }}</span>
                <div class="temp-controls">
                  <svg class="thermometer-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                    <path d="M14 14.76V3.5a2.5 2.5 0 0 0-5 0v11.26a4.5 4.5 0 1 0 5 0z"/>
                  </svg>
                  <button class="unit-toggle" @click="toggleUnit" :disabled="!weather">
                    °{{ unit.value === 'metric' ? 'C' : 'F' }}
                  </button>
                </div>
              </div>
              <div class="temp-range">High: 27 Low: 10</div>
              <div class="description">{{ weather ? weather.weather[0].description : 'Cloudy' }}</div>
              <div class="feels-like">Feels Like {{ weather ? Math.round(weather.main.feels_like) : '26' }}</div>
            </div>

            <div class="weather-icon">
              <img
                v-if="weather"
                :src="`https://openweathermap.org/img/wn/${weather.weather[0].icon}@4x.png`"
                :alt="weather.weather[0].description"
              />
              <svg v-else viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
                <path d="M12 3v1m0 16v1m9-9h-1M4 12H3m15.364 6.364l-.707-.707M6.343 6.343l-.707-.707m12.728 0l-.707.707M6.343 17.657l-.707.707M16 12a4 4 0 11-8 0 4 4 0 018 0z"/>
                <path d="M12 8.5a3.5 3.5 0 100 7 3.5 3.5 0 000-7z"/>
                <path d="M8 16.5c0-1.5 1.5-3 3.5-3s3.5 1.5 3.5 3"/>
              </svg>
            </div>
          </div>
        </div>

        <!-- City Name Section -->
        <div class="city-name-section">
          <div class="city-name">
            <svg class="location-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M21 10c0 7-9 13-9 13s-9-6-9-13a9 9 0 0 1 18 0z"/>
              <circle cx="12" cy="10" r="3"/>
            </svg>
            <span class="city-text">{{ weather ? weather.name : 'Manila' }},</span>
            <span class="country-text">{{ weather ? weather.sys.country : 'Philippines' }}</span>
          </div>
        </div>

        <!-- Additional Weather Info Section -->
        <div class="additional-info-section">
          <div class="info-row">
            <div class="info-item">
              <svg class="info-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <circle cx="12" cy="12" r="5"/>
                <path d="M12 1v6m0 6v6m11-7h-6M5 12H1"/>
              </svg>
              <div class="info-content">
                <div class="info-label">Sunrise</div>
                <div class="info-value">{{ weather ? '6:30 AM' : '6:30 AM' }}</div>
              </div>
            </div>
            <div class="info-item">
              <svg class="info-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M21 12.79A9 9 0 1 1 11.21 3 7 7 0 0 0 21 12.79z"/>
              </svg>
              <div class="info-content">
                <div class="info-label">Sunset</div>
                <div class="info-value">{{ weather ? '6:15 PM' : '6:15 PM' }}</div>
              </div>
            </div>
          </div>

          <div class="info-row">
            <div class="info-item">
              <svg class="info-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M9 12l2 2 4-4"/>
                <path d="M21 12c.552 0 1-.448 1-1V5c0-.552-.448-1-1-1H3c-.552 0-1 .448-1 1v6c0 .552.448 1 1 1"/>
                <path d="M3 21h18"/>
                <path d="M7 21V12"/>
                <path d="M17 21V12"/>
              </svg>
              <div class="info-content">
                <div class="info-label">UV Index</div>
                <div class="info-value">{{ weather ? 'Moderate' : 'Moderate' }}</div>
              </div>
            </div>
            <div class="info-item">
              <svg class="info-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
                <path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z"/>
                <circle cx="12" cy="12" r="3"/>
              </svg>
              <div class="info-content">
                <div class="info-label">Visibility</div>
                <div class="info-value">{{ weather ? '10 km' : '10 km' }}</div>
              </div>
            </div>
          </div>

          <div class="last-updated">
            <svg class="time-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <circle cx="12" cy="12" r="10"/>
              <polyline points="12,6 12,12 16,14"/>
            </svg>
            <span>Last updated: {{ new Date().toLocaleTimeString() }}</span>
          </div>
        </div>
      </div>

      <!-- Right Column - Details & Other Cities -->
      <div class="right-column">
        <!-- Weather Details Cards -->
        <div class="weather-details">
          <div class="detail-card">
            <div class="detail-title">Humidity</div>
            <div class="detail-value" v-if="weather">{{ weather.main.humidity }}%</div>
            <div class="detail-value" v-else>65%</div>
          </div>

          <div class="detail-card">
            <div class="detail-title">Wind</div>
            <div class="detail-value" v-if="weather">{{ weather.wind.speed }} m/s</div>
            <div class="detail-value" v-else>5.2 m/s</div>
          </div>

          <div class="detail-card">
            <div class="detail-title">Pressure</div>
            <div class="detail-value" v-if="weather">{{ weather.main.pressure }} hPa</div>
            <div class="detail-value" v-else>1013 hPa</div>
          </div>
        </div>

        <!-- Other Cities Section -->
        <div class="other-cities-section">
          <div class="section-title">Other Cities</div>

          <div class="city-cards">
            <div class="city-card" @click="fetchWeather('Manila')">
              <div class="city-temp">26°</div>
              <svg class="city-weather-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
                <path d="M12 3v1m0 16v1m9-9h-1M4 12H3m15.364 6.364l-.707-.707M6.343 6.343l-.707-.707m12.728 0l-.707.707M6.343 17.657l-.707.707M16 12a4 4 0 11-8 0 4 4 0 018 0z"/>
                <path d="M12 8.5a3.5 3.5 0 100 7 3.5 3.5 0 000-7z"/>
                <path d="M8 16.5c0-1.5 1.5-3 3.5-3s3.5 1.5 3.5 3"/>
              </svg>
              <div class="city-name-small">Manila, PH</div>
            </div>

            <div class="city-card" @click="fetchWeather('Tokyo')">
              <div class="city-temp">14°</div>
              <svg class="city-weather-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
                <path d="M7 16.3c2.2 0 4-1.83 4-4.05 0-1.16-.57-2.26-1.71-3.19S7.29 6.75 7 5.3c-.29 1.45-1.14 2.84-2.29 3.76S3 11.1 3 12.25c0 2.22 1.8 4.05 4 4.05z"/>
                <path d="M12.56 6.6A10.97 10.97 0 0 0 14 3.02c.5 2.5 2 4.9 4 6.5s3 3.5 3 5.5a6.98 6.98 0 0 1-11.91 4.97"/>
              </svg>
              <div class="city-name-small">Tokyo, JP</div>
            </div>

            <div class="city-card" @click="fetchWeather('New York')">
              <div class="city-temp">8°</div>
              <svg class="city-weather-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
                <path d="M20 16.2A4.5 4.5 0 0 0 19.1 14a3.5 3.5 0 0 0-5.8 0 3.5 3.5 0 0 0-5.8 0A4.5 4.5 0 0 0 7 16.2"/>
                <path d="M12 12v8"/>
                <path d="M12 4v4"/>
              </svg>
              <div class="city-name-small">New York, US</div>
            </div>

            <div class="city-card" @click="fetchWeather('London')">
              <div class="city-temp">11°</div>
              <svg class="city-weather-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
                <path d="M7 16.3c2.2 0 4-1.83 4-4.05 0-1.16-.57-2.26-1.71-3.19S7.29 6.75 7 5.3c-.29 1.45-1.14 2.84-2.29 3.76S3 11.1 3 12.25c0 2.22 1.8 4.05 4 4.05z"/>
                <path d="M12.56 6.6A10.97 10.97 0 0 0 14 3.02c.5 2.5 2 4.9 4 6.5s3 3.5 3 5.5a6.98 6.98 0 0 1-11.91 4.97"/>
              </svg>
              <div class="city-name-small">London, UK</div>
            </div>

            <!-- Additional cities shown when "Show all" is clicked -->
            <div v-if="showAll" class="city-card" @click="fetchWeather('Paris')">
              <div class="city-temp">10°</div>
              <svg class="city-weather-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
                <path d="M12 3v1m0 16v1m9-9h-1M4 12H3m15.364 6.364l-.707-.707M6.343 6.343l-.707-.707m12.728 0l-.707.707M6.343 17.657l-.707.707M16 12a4 4 0 11-8 0 4 4 0 018 0z"/>
                <path d="M12 8.5a3.5 3.5 0 100 7 3.5 3.5 0 000-7z"/>
                <path d="M8 16.5c0-1.5 1.5-3 3.5-3s3.5 1.5 3.5 3"/>
              </svg>
              <div class="city-name-small">Paris, FR</div>
            </div>

            <div v-if="showAll" class="city-card" @click="fetchWeather('Sydney')">
              <div class="city-temp">22°</div>
              <svg class="city-weather-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
                <path d="M12 3v1m0 16v1m9-9h-1M4 12H3m15.364 6.364l-.707-.707M6.343 6.343l-.707-.707m12.728 0l-.707.707M6.343 17.657l-.707.707M16 12a4 4 0 11-8 0 4 4 0 018 0z"/>
                <path d="M12 8.5a3.5 3.5 0 100 7 3.5 3.5 0 000-7z"/>
                <path d="M8 16.5c0-1.5 1.5-3 3.5-3s3.5 1.5 3.5 3"/>
              </svg>
              <div class="city-name-small">Sydney, AU</div>
            </div>

            <div v-if="showAll" class="city-card" @click="fetchWeather('Dubai')">
              <div class="city-temp">28°</div>
              <svg class="city-weather-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
                <path d="M12 3v1m0 16v1m9-9h-1M4 12H3m15.364 6.364l-.707-.707M6.343 6.343l-.707-.707m12.728 0l-.707.707M6.343 17.657l-.707.707M16 12a4 4 0 11-8 0 4 4 0 018 0z"/>
                <path d="M12 8.5a3.5 3.5 0 100 7 3.5 3.5 0 000-7z"/>
                <path d="M8 16.5c0-1.5 1.5-3 3.5-3s3.5 1.5 3.5 3"/>
              </svg>
              <div class="city-name-small">Dubai, AE</div>
            </div>

            <div v-if="showAll" class="city-card" @click="fetchWeather('Singapore')">
              <div class="city-temp">29°</div>
              <svg class="city-weather-icon" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5">
                <path d="M7 16.3c2.2 0 4-1.83 4-4.05 0-1.16-.57-2.26-1.71-3.19S7.29 6.75 7 5.3c-.29 1.45-1.14 2.84-2.29 3.76S3 11.1 3 12.25c0 2.22 1.8 4.05 4 4.05z"/>
                <path d="M12.56 6.6A10.97 10.97 0 0 0 14 3.02c.5 2.5 2 4.9 4 6.5s3 3.5 3 5.5a6.98 6.98 0 0 1-11.91 4.97"/>
              </svg>
              <div class="city-name-small">Singapore, SG</div>
            </div>
          </div>

          <button class="show-all-btn" @click="showAll = !showAll">
            {{ showAll ? 'Show less' : 'Show all' }}
          </button>
        </div>
      </div>
    </div>

    <!-- Error Message -->
    <div v-if="error" class="error">
      {{ error }}
    </div>

    <!-- API Key Warning -->
    <div v-if="API_KEY === 'YOUR_API_KEY_HERE'" class="api-key-warning">
      <p>⚠️ <strong>API Key Required:</strong> Please get a free API key from <a href="https://openweathermap.org/api" target="_blank">OpenWeatherMap</a> and replace 'YOUR_API_KEY_HERE' in the code. If you prefer, the app will automatically fall back to Open-Meteo (free, no API key needed).</p>
    </div>
  </div>
</template>
