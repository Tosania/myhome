<template>
  <div class="weather" v-if="weatherData.adCode.city && weatherData.weather.weather">
    <span>{{ weatherData.adCode.city }}&nbsp;</span>
    <span>{{ weatherData.weather.weather }}&nbsp;</span>
    <span>{{ weatherData.weather.temperature }}℃</span>
    <span class="sm-hidden">
      &nbsp;{{
        weatherData.weather.winddirection?.endsWith("风")
          ? weatherData.weather.winddirection
          : weatherData.weather.winddirection + "风"
      }}&nbsp;
    </span>
    <span class="sm-hidden">{{ weatherData.weather.windpower }}&nbsp;级</span>
  </div>
  <div class="weather" v-else>
    <span>{{ weatherStatus.message }}</span>
  </div>
</template>

<script setup>
import { getAdcode, getAmapGeocode, getWeather, getOtherWeather } from "@/api";
import { Error } from "@icon-park/vue-next";

// 高德开发者 Key
const mainKey = import.meta.env.VITE_WEATHER_KEY;
const configuredCity = import.meta.env.VITE_WEATHER_CITY?.trim();
const WEATHER_CACHE_KEY = "tosania-weather-cache";
const WEATHER_CACHE_MAX_AGE = 1000 * 60 * 30;

// 天气数据
const weatherData = reactive({
  adCode: {
    city: null, // 城市
    adcode: null, // 城市编码
  },
  weather: {
    weather: null, // 天气现象
    temperature: null, // 实时气温
    winddirection: null, // 风向描述
    windpower: null, // 风力级别
  },
});

const weatherStatus = reactive({
  message: "天气加载中",
});

// 取出天气平均值
const getTemperature = (min, max) => {
  try {
    // 计算平均值并四舍五入
    const average = (Number(min) + Number(max)) / 2;
    return Math.round(average);
  } catch (error) {
    console.error("计算温度出现错误：", error);
    return "NaN";
  }
};

const setWeatherData = ({ city, adcode, weather, temperature, winddirection, windpower }) => {
  weatherData.adCode = {
    city: city || "未知地区",
    adcode: adcode || null,
  };
  weatherData.weather = {
    weather,
    temperature,
    winddirection,
    windpower,
  };
};

const saveWeatherCache = () => {
  localStorage.setItem(
    WEATHER_CACHE_KEY,
    JSON.stringify({
      timestamp: Date.now(),
      data: {
        city: weatherData.adCode.city,
        adcode: weatherData.adCode.adcode,
        ...weatherData.weather,
      },
    }),
  );
};

const loadWeatherCache = () => {
  try {
    const cache = JSON.parse(localStorage.getItem(WEATHER_CACHE_KEY));
    if (!cache?.data || Date.now() - cache.timestamp > WEATHER_CACHE_MAX_AGE) return false;
    setWeatherData(cache.data);
    return true;
  } catch {
    return false;
  }
};

const assertWeather = (data) => {
  const temperature = Number(data?.temperature);
  if (
    !data?.weather ||
    data.temperature === undefined ||
    data.temperature === null ||
    Number.isNaN(temperature)
  ) {
    throw new Error("天气数据不完整");
  }
  return data;
};

const normalizeAmapWeather = (result, city, adcode) => {
  if (result?.infocode !== "10000" || !result?.lives?.[0]) {
    throw new Error(result?.info || "高德天气查询失败");
  }
  const live = result.lives[0];
  return assertWeather({
    city: city || live.city,
    adcode: adcode || live.adcode,
    weather: live.weather,
    temperature: live.temperature,
    winddirection: live.winddirection,
    windpower: live.windpower,
  });
};

const getCityAdcode = async (city) => {
  const result = await getAmapGeocode(mainKey, city);
  const geocode = result?.geocodes?.[0];
  if (result?.infocode !== "10000" || !geocode?.adcode) {
    throw new Error(result?.info || "城市编码查询失败");
  }
  return {
    city: Array.isArray(geocode.city) ? geocode.province : geocode.city || geocode.province || city,
    adcode: geocode.adcode,
  };
};

const getAmapWeatherByConfiguredCity = async () => {
  const location = await getCityAdcode(configuredCity);
  const result = await getWeather(mainKey, location.adcode);
  return normalizeAmapWeather(result, location.city, location.adcode);
};

const getAmapWeatherByIp = async () => {
  const adCode = await getAdcode(mainKey);
  if (adCode?.infocode !== "10000" || !adCode?.adcode) {
    throw new Error(adCode?.info || "IP 地区查询失败");
  }
  const result = await getWeather(mainKey, adCode.adcode);
  return normalizeAmapWeather(result, adCode.city, adCode.adcode);
};

const normalizeOtherWeather = (result) => {
  const data = result?.result;
  const city = data?.city?.City || data?.city?.city || configuredCity || "未知地区";
  const condition = data?.condition || {};
  return assertWeather({
    city,
    adcode: data?.city?.cityId,
    weather: condition.day_weather || condition.weather,
    temperature:
      condition.degree ?? getTemperature(condition.min_degree, condition.max_degree),
    winddirection: condition.day_wind_direction || condition.wind_direction,
    windpower: condition.day_wind_power || condition.wind_power,
  });
};

// 获取天气数据
const getWeatherData = async () => {
  const hasCache = loadWeatherCache();
  const providers = [];
  if (mainKey && configuredCity) providers.push(getAmapWeatherByConfiguredCity);
  if (mainKey) providers.push(getAmapWeatherByIp);
  providers.push(async () => normalizeOtherWeather(await getOtherWeather()));

  try {
    for (const provider of providers) {
      try {
        const data = await provider();
        setWeatherData(data);
        saveWeatherCache();
        return;
      } catch (error) {
        console.warn("天气接口尝试失败:", error);
      }
    }
    throw new Error("全部天气接口均不可用");
  } catch (error) {
    console.error("天气信息获取失败:", error);
    weatherStatus.message = hasCache ? "天气数据暂未更新" : "天气数据暂不可用";
    if (!hasCache) onError("天气信息获取失败");
  }
};

// 报错信息
const onError = (message) => {
  ElMessage({
    message,
    icon: h(Error, {
      theme: "filled",
      fill: "#efefef",
    }),
  });
  console.error(message);
};

onMounted(() => {
  // 调用获取天气
  getWeatherData();
});
</script>
