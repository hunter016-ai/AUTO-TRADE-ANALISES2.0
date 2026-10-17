# AUTO-TRADE-ANALISES2.0


// This Pine Script® code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © hryash014


 //@version=6
indicator("AUTO TRADE 2.0 | XAUUSD Scalping", overlay=true, format=format.price, max_lines_count=20, max_labels_count=20)

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 1. INPUTS
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
groupTrend = "1. Trend"
factor     = input.float(3.0, "Supertrend Factor", minval=0.1, step=0.1, group=groupTrend)
atrPeriod  = input.int(12, "Supertrend ATR Period", minval=1, group=groupTrend)
emaLength  = input.int(200, "EMA Trend Filter", minval=1, group=groupTrend)

groupFilter = "2. Filters"
adxThreshold = input.float(18.0, "Minimum ADX", minval=1, group=groupFilter)
useRSI       = input.bool(true, "Use RSI Filter", group=groupFilter)
useADX       = input.bool(true, "Use ADX Filter", group=groupFilter)
useHTF       = input.bool(true, "Use Higher Timeframe Filter", group=groupFilter)
htfTf        = input.timeframe("15", "Higher Timeframe", group=groupFilter)
htfEmaLen    = input.int(200, "HTF EMA Length", minval=1, group=groupFilter)

groupRisk = "3. Risk Management"
rr         = input.float(1.5, "Risk:Reward", minval=0.5, step=0.1, group=groupRisk)
slPadding  = input.float(1.0, "SL ATR Buffer", minval=0, step=0.1, group=groupRisk)
swingBars  = input.int(3, "Swing Lookback", minval=2, maxval=20, group=groupRisk)

groupVisual = "4. Visuals"
showDashboard = input.bool(true, "Show Dashboard", group=groupVisual)
showLevels    = input.bool(true, "Show SL/TP Lines", group=groupVisual)
textSizeInput = input.string("Small", "Dashboard Size",
     options=["Tiny", "Small", "Normal"], group=groupVisual)

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 2. INDICATORS
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
[st, direction] = ta.supertrend(factor, atrPeriod)
ema200 = ta.ema(close, emaLength)
atrVal = ta.atr(14)
rsiVal = ta.rsi(close, 14)
[diPlus, diMinus, adxVal] = ta.dmi(14, 14)

// Previous COMPLETED HTF candle values.
// This avoids using a still-forming HTF candle.
htfClose = request.security(syminfo.tickerid, htfTf,
     close[1], lookahead=barmerge.lookahead_on)
htfEma = request.security(syminfo.tickerid, htfTf,
     ta.ema(close, htfEmaLen)[1], lookahead=barmerge.lookahead_on)

htfLongOK  = not useHTF or htfClose > htfEma
htfShortOK = not useHTF or htfClose < htfEma

// Supertrend direction: negative = bullish, positive = bearish.
longFlip  = direction < 0 and direction[1] > 0
shortFlip = direction > 0 and direction[1] < 0

trendLong  = close > ema200
trendShort = close < ema200

rsiLongOK  = not useRSI or rsiVal > 50
rsiShortOK = not useRSI or rsiVal < 50
adxOK      = not useADX or adxVal > adxThreshold

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 3. SIGNALS — CONFIRMED BAR ONLY
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
longSetup = longFlip and trendLong and rsiLongOK and adxOK and htfLongOK
shortSetup = shortFlip and trendShort and rsiShortOK and adxOK and htfShortOK

// Entry modeled at the confirmed candle close.
// Real fills can differ due to spread and slippage.
goLong  = barstate.isconfirmed and longSetup
goShort = barstate.isconfirmed and shortSetup

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 4. SL / TP
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
var float lastEntry = na
var float lastSL = na
var float lastTP = na
var float lastRSI = na
var float lastADX = na
var int lastTime = na
var string lastType = "WAIT"

var line slLine = na
var line tpLine = na
var label slLabel = na
var label tpLabel = na

if goLong
    float swingLow = ta.lowest(low, swingBars)
    float candidateSL = swingLow - atrVal * slPadding
    float risk = close - candidateSL

    if risk > syminfo.mintick
        lastEntry := close
        lastSL := candidateSL
        lastTP := close + risk * rr
        lastRSI := rsiVal
        lastADX := adxVal
        lastTime := time
        lastType := "LONG"

if goShort
    float swingHigh = ta.highest(high, swingBars)
    float candidateSL = swingHigh + atrVal * slPadding
    float risk = candidateSL - close

    if risk > syminfo.mintick
        lastEntry := close
        lastSL := candidateSL
        lastTP := close - risk * rr
        lastRSI := rsiVal
        lastADX := adxVal
        lastTime := time
        lastType := "SHORT"

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 5. CHART DISPLAY
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
color green = color.rgb(0, 220, 100)
color pink  = color.rgb(255, 50, 170)

plot(ema200, "EMA 200", color=color.white, linewidth=2)
plot(direction < 0 ? st : na, "Bullish Supertrend",
     color=color.green, style=plot.style_linebr, linewidth=2)
plot(direction > 0 ? st : na, "Bearish Supertrend",
     color=color.red, style=plot.style_linebr, linewidth=2)

plotshape(goLong and lastType == "LONG" and lastTime == time,
     title="BUY", style=shape.labelup, location=location.belowbar,
     color=green, text="BUY", textcolor=color.black, size=size.small)

plotshape(goShort and lastType == "SHORT" and lastTime == time,
     title="SELL", style=shape.labeldown, location=location.abovebar,
     color=pink, text="SELL", textcolor=color.white, size=size.small)

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 6. SL / TP LINES
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
bool newSignal = (goLong or goShort) and lastTime == time

if newSignal and showLevels
    line.delete(slLine)
    line.delete(tpLine)
    label.delete(slLabel)
    label.delete(tpLabel)

    slLine := line.new(bar_index, lastSL, bar_index + 15, lastSL,
         color=color.red, width=2)
    tpLine := line.new(bar_index, lastTP, bar_index + 15, lastTP,
         color=color.blue, width=2)

    slLabel := label.new(bar_index + 5, lastSL,
         "SL " + str.tostring(lastSL, format.mintick),
         style=label.style_label_up, color=color.red,
         textcolor=color.white, size=size.small)

    tpLabel := label.new(bar_index + 5, lastTP,
         "TP " + str.tostring(lastTP, format.mintick),
         style=label.style_label_down, color=color.blue,
         textcolor=color.white, size=size.small)

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 7. DASHBOARD
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
var table dash = table.new(position.bottom_right, 2, 7,
     bgcolor=color.new(color.black, 20), border_width=1)

txtSize = textSizeInput == "Tiny" ? size.tiny :
     textSizeInput == "Normal" ? size.normal : size.small

if barstate.islast and showDashboard
    color typeColor = lastType == "LONG" ? green :
         lastType == "SHORT" ? pink : color.gray

    table.cell(dash, 0, 0, "AUTO TRADE 2.0",
         text_color=color.yellow, text_size=txtSize)
    table.cell(dash, 1, 0, lastType,
         text_color=typeColor, text_size=txtSize)

    table.cell(dash, 0, 1, "Entry", text_color=color.white, text_size=txtSize)
    table.cell(dash, 1, 1, str.tostring(lastEntry, format.mintick),
         text_color=color.white, text_size=txtSize)

    table.cell(dash, 0, 2, "Stop Loss", text_color=color.red, text_size=txtSize)
    table.cell(dash, 1, 2, str.tostring(lastSL, format.mintick),
         text_color=color.red, text_size=txtSize)

    table.cell(dash, 0, 3, "Take Profit", text_color=color.aqua, text_size=txtSize)
    table.cell(dash, 1, 3, str.tostring(lastTP, format.mintick),
         text_color=color.aqua, text_size=txtSize)

    table.cell(dash, 0, 4, "Signal RSI", text_color=color.silver, text_size=txtSize)
    table.cell(dash, 1, 4, str.tostring(lastRSI, "#.0"),
         text_color=color.white, text_size=txtSize)

    table.cell(dash, 0, 5, "Signal ADX", text_color=color.silver, text_size=txtSize)
    table.cell(dash, 1, 5, str.tostring(lastADX, "#.0"),
         text_color=color.white, text_size=txtSize)

    table.cell(dash, 0, 6, "Risk : Reward", text_color=color.yellow, text_size=txtSize)
    table.cell(dash, 1, 6, "1 : " + str.tostring(rr, "#.##"),
         text_color=color.yellow, text_size=txtSize)

//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 8. ALERTS
//━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
if newSignal and lastType == "LONG"
    alert("AUTO TRADE 2.0 | BUY " + syminfo.ticker +
         " | Entry: " + str.tostring(lastEntry, format.mintick) +
         " | SL: " + str.tostring(lastSL, format.mintick) +
         " | TP: " + str.tostring(lastTP, format.mintick),
         alert.freq_once_per_bar_close)

if newSignal and lastType == "SHORT"
    alert("AUTO TRADE 2.0 | SELL " + syminfo.ticker +
         " | Entry: " + str.tostring(lastEntry, format.mintick) +
         " | SL: " + str.tostring(lastSL, format.mintick) +
         " | TP: " + str.tostring(lastTP, format.mintick),
         alert.freq_once_per_bar_close)
