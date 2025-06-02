<template>
   <div class="container">
      <h1>Online price change</h1>
      <span v-if="connection_error" class="error-msg">Connection Error!</span>
      <div class="page-content">
         <select name="currency" id="currency" @change="changeCurrency">
            <option value="BTCUSDT">Bitcoin</option>
            <option value="ETHUSDT">Ethereum</option>
         </select>
         <p v-if="loading" class="loading">
            <span class="loader"></span>
         </p>
         <div class="price-box">
            <img :src="currencyImg" alt="" />
            <div class="price-info">
               <h2>{{ currency === "BTCUSDT" ? "Bitcoin" : "Ethereum" }} / TetherUS</h2>
               <h2 :class="trendStatus">
                  {{ formattedPrice }}
                  <span>IRT</span>
               </h2>
            </div>
         </div>
      </div>
      <div class="chart-container" ref="chartContainer"></div>
   </div>
</template>

<script>
import { CandlestickSeries, createChart } from "lightweight-charts";
import axios from "axios";

export default {
   name: "HomeView",

   data() {
      return {
         connection_ready: false,
         connection_error: false,
         websocket: null,
         current_price: 0,
         prev_price: 0,
         currency: "BTCUSDT",
         loading: true,
         chart: null,
         candleSeries: null,
         currentCandle: null,
         interval: 5 * 60,
         lastCandleTime: 0,
         dollarIRT: 82000,
      };
   },

   methods: {
      init_chat() {
         var sockets_bay_url = `wss://stream.bybit.com/v5/public/spot`;
         this.websocket = new WebSocket(sockets_bay_url);

         this.websocket.onopen = this.onSocketOpen;
         this.websocket.onmessage = this.onSocketMessage;
         this.websocket.onerror = this.onSocketError;
      },

      onSocketOpen(evt) {
         this.connection_ready = true;
         const subscribeMessage = {
            op: "subscribe",
            args: [`tickers.${this.currency}`, `kline.5.${this.currency}`],
         };
         this.websocket.send(JSON.stringify(subscribeMessage));
         this.loading = false;
      },
      onSocketMessage(evt) {
         const msg = JSON.parse(evt.data);

         if (msg.topic === `tickers.${this.currency}` && msg.data) {
            this.prev_price = this.current_price;
            this.current_price = JSON.parse(evt.data).data
               ? JSON.parse(evt.data).data.lastPrice
               : 0;
         }
         if (msg.topic === `kline.5.${this.currency}` && msg.data) {
            const klineUpdate = msg.data[0];
            this.candleSeries.update({
               time: parseInt(klineUpdate.start) / 1000,
               open: parseFloat(klineUpdate.open) * this.dollarIRT,
               high: parseFloat(klineUpdate.high) * this.dollarIRT,
               low: parseFloat(klineUpdate.low) * this.dollarIRT,
               close: parseFloat(klineUpdate.close) * this.dollarIRT, // Live price is usually 'close' for current candle
            });
         }
      },

      setupChart() {
         this.chart = createChart(this.$refs.chartContainer, {
            layout: { background: { color: "#fff" }, textColor: "#000" },
            width: this.$refs.chartContainer.clientWidth,
            height: 400,
            timeScale: { timeVisible: true, secondsVisible: false },
         });
         this.candleSeries = this.chart.addSeries(CandlestickSeries);
      },

      onSocketError(evt) {
         this.connection_error = true;
      },

      changeCurrency(e) {
         this.loading = true;
         this.currency = e.target.value;
      },

      async fetchHistoricalKlineData() {
         try {
            const endpoint = "/v5/market/kline"; // For unified market data
            const params = {
               category: "spot", // Or 'spot' if you change wsBaseUrl to spot
               symbol: this.currency,
               interval: "5",
               limit: 300,
            };

            const response = await axios.get(`https://api.bybit.com${endpoint}`, {
               params,
            });
            const klineData = response.data.result.list;

            if (klineData && klineData.length > 0) {
               // Bybit returns newest data first, reverse it for Lightweight Charts
               const formattedData = klineData.reverse().map((k) => ({
                  time: parseInt(k[0]) / 1000, // Convert ms to seconds
                  open: parseFloat(k[1]) * this.dollarIRT,
                  high: parseFloat(k[2]) * this.dollarIRT,
                  low: parseFloat(k[3]) * this.dollarIRT,
                  close: parseFloat(k[4]) * this.dollarIRT,
               }));
               this.candleSeries.setData(formattedData);
            } else {
               console.warn("No historical kline data received.");
            }
         } catch (error) {
            console.error("Error fetching historical kline data:", error);
         }
      },
   },

   computed: {
      formattedPrice() {
         const formatter = new Intl.NumberFormat("en-US", {
            minimumFractionDigits: 0,
            maximumFractionDigits: 0,
         });
         if (this.current_price) {
            return formatter.format(this.current_price * this.dollarIRT);
         } else {
            return "0:00";
         }
      },

      trendStatus() {
         if (this.current_price < this.prev_price) return "bearish";
         else return "bullish";
      },

      currencyImg() {
         if (this.currency === "BTCUSDT") return "/src/assets/XTVCBTC--big.svg";
         else return "/src/assets/XTVCETH--big.svg";
      },
   },

   async mounted() {
      this.setupChart();
      await this.fetchHistoricalKlineData();
      this.init_chat();
   },

   beforeUnmount() {
      if (this.websocket) {
         this.websocket.close();
      }
   },

   watch: {
      async currency(newVal) {
         this.websocket.close();
         this.init_chat(newVal);
         await this.fetchHistoricalKlineData();
      },
   },
};
</script>

<style>
body {
   font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
   background-color: #242424;
}

.container {
   display: flex;
   justify-content: center;
   align-items: center;
   flex-direction: column;
}

h1 {
   text-align: center;
   color: #fff;
   margin-bottom: 5rem;
}

.w-full {
   width: 100%;
}

.price-box {
   display: flex;
   justify-content: center;
   align-items: center;
   gap: 2rem;
}

.price-box img {
   border-radius: 50%;
   width: 100px;
}

.price-box .price-info {
   display: flex;
   flex-direction: column;
   gap: 1rem;
}

.price-box .price-info h2 {
   color: #fff;
   margin: 0;
}

.price-box .price-info h2 span {
   font-size: 1rem;
   color: #fff;
}

.price-box .price-info h2.bearish {
   color: #e43636;
}

.price-box .price-info h2.bullish {
   color: #089232;
}

.error-msg {
   text-align: center;
   color: #fff;
   font-size: 2rem;
   margin-bottom: 3rem;
   font-weight: bold;
}

#currency {
   display: inline-flex;
   min-width: 25%;
   min-height: 2rem;
   margin-bottom: 3rem;
   background-color: #303030;
   color: #fff;
}

.page-content {
   display: flex;
   flex-direction: column;
   justify-content: center;
   position: relative;
   margin-bottom: 6rem;
}

.loading {
   text-align: center;
   color: #fff;
   font-weight: bold;
   position: absolute;
   top: -0.75rem;
   left: -0.75rem;
   width: calc(100% + 1.5rem);
   height: calc(100% + 1.5rem);
   background-color: #585858a2;
   border-radius: 0.75rem;
   margin: 0;
   display: flex;
   justify-content: center;
   align-items: center;
}

.loader {
   width: 48px;
   height: 48px;
   border-radius: 50%;
   position: relative;
   animation: rotate 1s linear infinite;
}

.loader::before {
   content: "";
   box-sizing: border-box;
   position: absolute;
   inset: 0px;
   border-radius: 50%;
   border: 5px solid #fff;
   animation: prixClipFix 2s linear infinite;
}

.chart-container {
   height: 350px;
   width: 90%;
}

@keyframes rotate {
   100% {
      transform: rotate(360deg);
   }
}

@keyframes prixClipFix {
   0% {
      clip-path: polygon(50% 50%, 0 0, 0 0, 0 0, 0 0, 0 0);
   }
   25% {
      clip-path: polygon(50% 50%, 0 0, 100% 0, 100% 0, 100% 0, 100% 0);
   }
   50% {
      clip-path: polygon(50% 50%, 0 0, 100% 0, 100% 100%, 100% 100%, 100% 100%);
   }
   75% {
      clip-path: polygon(50% 50%, 0 0, 100% 0, 100% 100%, 0 100%, 0 100%);
   }
   100% {
      clip-path: polygon(50% 50%, 0 0, 100% 0, 100% 100%, 0 100%, 0 0);
   }
}
</style>
