<!-- links:-->
<!-- voir : https://layercake.graphics/example-ssr/Line&ndash;&gt;-->
<!-- https://www.visualcinnamon.com/2016/06/fun-data-visualizations-svg-gooey-effect/-->

<script>
    import * as LightweightCharts from 'lightweight-charts';
    import {createChart} from 'lightweight-charts';

    document.body.style.position = 'relative';

    var container = document.createElement('div');
    document.body.appendChild(container);

    var width = 800;
    var height = 600;

    let chart = createChart(container, {
        width: width,
        height: height,
        timeScale: {
            timeVisible: true,
            secondsVisible: false,
        },
        rightPriceScale: {
            visible: true,
            borderColor: 'rgba(197, 203, 206, 1)',
        }
    });

    var toolTipWidth = 100;
    var toolTipHeight = 80;
    var toolTipMargin = 15;

    chart.timeScale().subscribeVisibleTimeRangeChange(event => {
        //console.log("time range changed: " + JSON.stringify(event))
    })
    chart.timeScale().subscribeVisibleLogicalRangeChange(event => {
        //console.log("logical range changed: " + JSON.stringify(event))
    })

    let lineSeriesPERP = chart.addCandlestickSeries();
    let volumeSeries = chart.addHistogramSeries({
        color: '#26a69a',
        priceFormat: {
            type: 'volume',
        },
        priceScaleId: 'total',
        scaleMargins: {
            top: 0.8,
            bottom: 0,
        }
    });
    let volumePosSeries = chart.addHistogramSeries({
        color: '#26a69a',
        priceFormat: {
            type: 'volume',
        },
        priceScaleId: 'buy',
        scaleMargins: {
            top: 0.8,
            bottom: 0,
        },
    });

    var toolTip = document.createElement('div');
    toolTip.className = 'floating-tooltip-2';
    container.appendChild(toolTip);

    // update tooltip
    chart.subscribeCrosshairMove(function (param) {
        if (!param.time || param.point.x < 0 || param.point.x > width || param.point.y < 0 || param.point.y > height) {
            toolTip.style.display = 'none';
            return;
        }

        var dateStr = new Date(param.time*1000).toLocaleTimeString();

        toolTip.style.display = 'block';
        var price = param.seriesPrices.get(lineSeriesPERP).close;
        var volume = param.seriesPrices.get(volumeSeries);
        var volumeBuy = Math.round(param.seriesPrices.get(volumePosSeries)*10)/10;
        var volumeSell = Math.round((volume - volumeBuy)*10)/10;

        toolTip.innerHTML = '<div style="color: rgba(255, 70, 70, 1)">ETH-PERP</div>' +
            '<div style="font-size: 24px; margin: 4px 0px">' + price + '</div>' +
            '<div>time: ' + dateStr  + '</div>' +
            '<div>volume (buy): ' + volumeBuy  + '</div>'+
            '<div>volume (sell): ' + volumeSell  + '</div>';

        var y = param.point.y;

        var left = param.point.x + toolTipMargin;
        if (left > width - toolTipWidth) {
            left = param.point.x - toolTipMargin - toolTipWidth;
        }

        var top = y + toolTipMargin;
        if (top > height - toolTipHeight) {
            top = y - toolTipHeight - toolTipMargin;
        }

        toolTip.style.left = left + 'px';
        toolTip.style.top = top + 'px';
    });

    let markers = []
    let bidPrice = 0
    let askPrice = 0
    lineSeriesPERP.setMarkers(markers)
    const socket = new WebSocket("wss://ftx.com/ws/");
    const symbol = "ETH-PERP"
    socket.addEventListener('open', function (event) {
        socket.send('{"op": "subscribe", "channel": "trades", "market": "'+symbol+'"}')
        socket.send('{"op": "subscribe", "channel": "orderbook", "market": "'+symbol+'"}')
        socket.send('{"op": "subscribe", "channel": "ticker", "market": "'+symbol+'"}')
    });

    let lastTime = 0
    let liquidations = new Map([])
    let volumes = new Map([])
    let volumesPos = new Map([])
    let pointsInBar = []
    let open = null
    let close = null
    let lastPrice = null
    let volume = 0
    let volumePos = 0
    let veryLastPrice = 0

    let bids = new Map([])
    let asks = new Map([])

    let createdLines = []

    function initOrderbook(data) {

        bids = new Map([])
        asks = new Map([])

        data.bids.forEach(bid => bids.set(bid[0], bid[1]))
        data.asks.forEach(bid => asks.set(bid[0], bid[1]))

        drawOrderBook()
    }

    function updateOrderbook(data) {

        data.bids.forEach(bid => bids.set(bid[0], bid[1]))
        data.asks.forEach(bid => asks.set(bid[0], bid[1]))

        for (const [price, size] of bids.entries()) {
            if (price > bidPrice || size === 0) {
                bids.delete(price)
            }
        }
        for (const [price, size] of asks.entries()) {
            if (price < askPrice || size === 0) {
                asks.delete(price)
            }
        }

        drawOrderBook()
    }

    function removeLines() {
        createdLines.forEach(line => lineSeriesPERP.removePriceLine(line))
        createdLines = []
    }

    function drawOrderBook() {
        let maxBid = Math.max(...bids.values())
        let maxAsk = Math.max(...asks.values())
        let max = Math.max(maxBid, maxAsk)
        removeLines()

        for (const [price, size] of bids.entries()) {
            if (size > 100000) {
                let ratio = 0.1+(size / max)*0.9
                let label = size

                let line = {
                    price: price,
                    color: "rgba(0,250,0," + ratio + ")",
                    lineWidth: 0.5+ (3 * ratio),
                    axisLabelVisible: (size >= max / 10),
                    title: label,
                    lineStyle: LightweightCharts.LineStyle.Solid
                }
                let addedLine = lineSeriesPERP.createPriceLine(line)
                createdLines.push(addedLine)
            }
        }
        for (const [price, size] of asks.entries()) {
            if (size > 100000) {
                let ratio = 0.1+(size / max)*0.9
                let label = size

                let line = {
                    price: price,
                    color: "rgba(250,0,0," + ratio + ")",
                    lineWidth: 0.5+ (3 * ratio),
                    axisLabelVisible: (size >= max / 3),
                    title: label,
                    lineStyle: LightweightCharts.LineStyle.Solid
                }
                let addedLine = lineSeriesPERP.createPriceLine(line)
                createdLines.push(addedLine)
            }
        }
    }

    function updateTicker(data) {
        bidPrice = data.bid
        askPrice = data.ask
    }

    function handleLiquidations(point, date) {
        if (point.liquidation) {
            let totalLiquidation = liquidations.get(date)
            if (totalLiquidation === undefined) {
                totalLiquidation = 0
            }
            liquidations.set(date, totalLiquidation + point.size)

            let markers = []
            for (const [time, size] of liquidations.entries()) {
                markers.push({
                    time: time,
                    position: 'aboveBar',
                    color: '#f68410',
                    shape: 'circle',
                    text: "" + Math.round(size * 100) / 100
                })
            }
            lineSeriesPERP.setMarkers(markers)
        }
    }

    socket.addEventListener('message', function (event) {
        let data = JSON.parse(event.data)
        let points = data.data
        let type = data.type
        let channel = data.channel

        if (channel === "orderbook" && type === "partial") {
            initOrderbook(data.data)
        }
        if (channel === "orderbook" && type === "update") {
            updateOrderbook(data.data)
        }
        if (channel === "ticker" && type === "update") {
            updateTicker(data.data)
        }

        if (channel === "trades" && type === "update") {
            let market = data.market

            points.forEach(point => {

                veryLastPrice = point.price

                let grid = 10
                let timeSeconds = Date.parse(point.time) / 1000
                let time = Math.trunc(timeSeconds/grid)*grid

                // let side = point.side;
                // let size = point.size;
                if (open == null) {
                    open = point.price
                    lastPrice = open
                }

                handleLiquidations(point, time);

                if (time === lastTime) {
                    pointsInBar.push(point.price)
                    lastPrice = point.price
                    volume += point.size
                    if (point.side === "buy") {
                        volumePos += point.size
                    }

                } else {
                    pointsInBar = [lastPrice, point.price]
                    lastTime = time;
                    open = lastPrice;
                    volume = 0
                    volumePos = 0
                }

                let newPoint = {
                    time: time,
                    open: open,
                    high: pointsInBar.reduce((a, b) => Math.max(a, b)),
                    low: pointsInBar.reduce((a, b) => Math.min(a, b)),
                    close: lastPrice
                }
                lineSeriesPERP.update(newPoint)

                let newVolume = {
                    time: time,
                    value: volume,
                    color: 'rgba(149,0,166,0.2)'
                }
                volumeSeries.update(newVolume)

                let newVolumePos = {
                    time: time,
                    value: volumePos,
                    color: 'rgba(0,141,166,0.27)'
                }
                volumePosSeries.update(newVolumePos)
            })
        }
    });

</script>

<!--TODO:

- corriger la gestion de l'orderbook: bid et ask séparés, effacer ceux qui sont en décalage avec le bid et ask
- séparer le ticker de l'orderbook

- pbs de perfs
- liquidations: separer les buy et sells

- ordre de vente en click
- tool pour voir le pourcentage fees

- s'alimenter sur mon serveur en fonction zoom: https://jsfiddle.net/TradingView/fg7yez2s/
- forker la lib ?

-->

<style>

    .floating-tooltip-2 {
        width: 96px;
        height: 80px;
        position: absolute;
        display: none;
        padding: 8px;
        box-sizing: border-box;
        font-size: 12px;
        color: #131722;
        background-color: rgba(255, 255, 255, 1);
        text-align: left;
        z-index: 1000;
        top: 12px;
        left: 12px;
        pointer-events: none;
        border: 1px solid rgba(255, 70, 70, 1);
        border-radius: 2px;
    }

</style>
