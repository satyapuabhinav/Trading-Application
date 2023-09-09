# Trading-Application
<?0xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>CFBundleVersion</key>
        <string>1.42.20</string>
        <string>1.42.21</string>
	<key>CFBundleName</key>
	<string>Qt Bitcoin Trader</string>
	<key>CFBundleIconFile</key>
IDI_ICON1 ICON "Icon.ico"
1 VERSIONINFO
 FILEVERSION 1,4,2,20
 PRODUCTVERSION 1,4,2,20
 FILEVERSION 1,4,2,21
 PRODUCTVERSION 1,4,2,21
BEGIN
    BLOCK "StringFileInfo"
    BEGIN
        BLOCK "040904b0"
        BEGIN
            VALUE "CompanyName", "Centrabit AG"
            VALUE "FileDescription", "Qt Bitcoin Trader"
            VALUE "FileVersion", "1.42.20"
            VALUE "FileVersion", "1.42.21"
            VALUE "InternalName", "Qt Bitcoin Trader"
            VALUE "LegalCopyright", "Ighor July (C) 2023"
            VALUE "OriginalFilename", "QtBitcoinTrader.exe"
            VALUE "ProductName", "Qt Bitcoin Trader Application"
            VALUE "ProductVersion", "1.42.20"
            VALUE "ProductVersion", "1.42.21"
        END
    END
	BLOCK "VarFileInfo"
	BEGIN
	VALUE "Translation", 0x0409, 0x04B0
	END
END
#include <QJsonDocument>
#include <QJsonObject>
#include <QJsonArray>
#include "timesync.h"
#include "iniengine.h"
#include "exchange_poloniex.h"
static const int c_detlaHistoryTime = 27 * 24 * 60 * 60;
Exchange_Poloniex::Exchange_Poloniex(const QByteArray &pRestSign, const QByteArray &pRestKey)
    : Exchange(),
      isFirstAccInfo(true),
      lastTradeId(0),
      lastHistoryId(0),
      julyHttp(nullptr),
      depthAsks(nullptr),
      depthBids(nullptr),
      lastDepthAsksMap(),
      lastDepthBidsMap()
{
    clearHistoryOnCurrencyChanged = false;
    calculatingFeeMode = 1;
    baseValues.exchangeName = "Poloniex";
    baseValues.currentPair.name = "ETH/BTC";
    baseValues.currentPair.setSymbol("ETH/BTC");
    baseValues.currentPair.currRequestPair = "BTC_ETH";
    baseValues.currentPair.priceDecimals = 8;
    minimumRequestIntervalAllowed = 500;
    minimumRequestTimeoutAllowed = 10000;
    baseValues.currentPair.priceMin = qPow(0.1, baseValues.currentPair.priceDecimals);
    baseValues.currentPair.tradeVolumeMin = 0.00000001;
    baseValues.currentPair.tradePriceMin = 0.00000001;
    forceDepthLoad = false;
    tickerOnly = false;
    setApiKeySecret(pRestKey, pRestSign);
    currencyMapFile = "Poloniex";
    defaultCurrencyParams.currADecimals = 8;
    defaultCurrencyParams.currBDecimals = 8;
    defaultCurrencyParams.currABalanceDecimals = 8;
    defaultCurrencyParams.currBBalanceDecimals = 8;
    defaultCurrencyParams.priceDecimals = 8;
    defaultCurrencyParams.priceMin = qPow(0.1, baseValues.currentPair.priceDecimals);
    supportsLoginIndicator = false;
    supportsAccountVolume = false;
    connect(this, &Exchange::threadFinished, this, &Exchange_Poloniex::quitThread, Qt::DirectConnection);
}
Exchange_Poloniex::~Exchange_Poloniex()
{
}
void Exchange_Poloniex::quitThread()
{
    clearValues();
    
        delete depthAsks;
    
        delete depthBids;
    
        delete julyHttp;
}
void Exchange_Poloniex::clearVariables()
{
    isFirstAccInfo = true;
    lastTradeId = 0;
    Exchange::clearVariables();
    lastHistory.clear();
    lastOrders.clear();
    reloadDepth();
}
void Exchange_Poloniex::clearValues()
{
    clearVariables();
    if (julyHttp)
        julyHttp->clearPendingData();
}
void Exchange_Poloniex::reloadDepth()
{
    lastDepthBidsMap.clear();
    lastDepthAsksMap.clear();
    lastDepthData.clear();
    Exchange::reloadDepth();
}
void Exchange_Poloniex::dataReceivedAuth(const QByteArray& data, int reqType, int pairChangeCount)
{
    if (pairChangeCount != m_pairChangeCount)
        return;
    if (debugLevel)
        logThread->writeLog("RCV: " + data);
    if (data.size() && data.at(0) == QLatin1Char('<'))
        return;
    bool success = !data.startsWith("{\"error\":");//{"error":"Invalid command."}
    QString errorString;
    if (!success)
    {
        errorString = getMidData("{\"error\":\"", "\"", &data);
        if (debugLevel)
            logThread->writeLog("Invalid data:" + data, 2);
    }
    else switch (reqType)
    {
    case 103: //ticker
    {
        QJsonObject ticker = QJsonDocument::fromJson(data).object();
        double tickerHigh =  ticker.value("high").toString().toDouble();
        if (tickerHigh > 0.0 && !qFuzzyCompare(tickerHigh, lastTickerHigh))
        {
            IndicatorEngine::setValue(baseValues.exchangeName, baseValues.currentPair.symbol, "High", tickerHigh);
            lastTickerHigh = tickerHigh;
        }
        double tickerLow = ticker.value("low").toString().toDouble();
        if (tickerLow > 0.0 && !qFuzzyCompare(tickerLow, lastTickerLow))
        {
            IndicatorEngine::setValue(baseValues.exchangeName, baseValues.currentPair.symbol, "Low", tickerLow);
            lastTickerLow = tickerLow;
        }
        double tickerSell = ticker.value("bid").toString().toDouble();
        if (tickerSell > 0.0 && !qFuzzyCompare(tickerSell, lastTickerSell))
        {
            IndicatorEngine::setValue(baseValues.exchangeName, baseValues.currentPair.symbol, "Sell", tickerSell);
            lastTickerSell = tickerSell;
        }
        double tickerBuy = ticker.value("ask").toString().toDouble();
        if (tickerBuy > 0.0 && !qFuzzyCompare(tickerBuy, lastTickerBuy))
        {
            IndicatorEngine::setValue(baseValues.exchangeName, baseValues.currentPair.symbol, "Buy", tickerBuy);
            lastTickerBuy = tickerBuy;
        }
        double tickerVolume = ticker.value("quantity").toString().toDouble();
        if (tickerVolume > 0.0 && !qFuzzyCompare(tickerVolume, lastTickerVolume))
        {
            IndicatorEngine::setValue(baseValues.exchangeName, baseValues.currentPair.symbol, "Volume", tickerVolume);
            lastTickerVolume = tickerVolume;
        }
        double tickerLast = ticker.value("close").toString().toDouble();
        if (tickerLast > 0.0 && !qFuzzyCompare(tickerLast, lastTickerLast))
        {
            IndicatorEngine::setValue(baseValues.exchangeName, baseValues.currentPair.symbol, "Last", tickerLast);
            lastTickerLast = tickerLast;
        }
    }
    break;//ticker
    case 109: //trades
        if (data.size() > 10)
        {
            QJsonArray tradeList = QJsonDocument::fromJson(data).array();
            qint64 time10Min = TimeSync::getTimeT() - 600;
            auto* newTradesItems = new QList<TradesItem>;
            qint64 packetLastTradeId = 0;
            for (int n = 0; n < tradeList.size(); ++n)
            {
                QJsonObject tradeData = tradeList.at(n).toObject();
                QString tradeIdStr = tradeData.value("id").toString();
                qint64 tradeId = tradeIdStr.toLongLong();
                if (tradeId <= lastTradeId)
                    break;
                if (!packetLastTradeId)
                    packetLastTradeId = tradeId;
                TradesItem newItem;
                newItem.date = tradeData.value("ts").toVariant().toLongLong() / 1000;
                if (newItem.date < time10Min)
                    break;
                newItem.amount    = tradeData.value("quantity").toString().toDouble();
                newItem.price     = tradeData.value("price").toString().toDouble();
                newItem.symbol    = baseValues.currentPair.symbol;
                newItem.orderType = tradeData.value("takerSide").toString() == "BUY" ? -1 : 1;
                if (newItem.isValid())
                    newTradesItems->prepend(newItem);
                else if (debugLevel)
                    logThread->writeLog("Invalid trades fetch data id:" + QByteArray::number(tradeId), 2);
            }
            if (packetLastTradeId)
                lastTradeId = packetLastTradeId;
            if (!newTradesItems->empty())
                emit addLastTrades(baseValues.currentPair.symbol, newTradesItems);
            else
                delete newTradesItems;
        }
        break;//trades
    case 111: //depth
    {
        emit depthRequestReceived();
        if (data != lastDepthData)
        {
            lastDepthData = data;
            depthAsks = new QList<DepthItem>;
            depthBids = new QList<DepthItem>;
            QJsonObject depht = QJsonDocument::fromJson(data).object();
            QJsonArray asksList = depht.value("asks").toArray();
            QJsonArray bidsList = depht.value("bids").toArray();
            QMap<double, double> currentAsksMap;
            QMap<double, double> currentBidsMap;
            double groupedPrice = 0.0;
            double groupedVolume = 0.0;
            int rowCounter = 0;
            for (int n = 1; n < asksList.size(); n += 2)
            {
                if (baseValues.depthCountLimit && rowCounter >= baseValues.depthCountLimit)
                    break;
                double priceDouble = asksList.at(n - 1).toString().toDouble();
                double amount      = asksList.at(n    ).toString().toDouble();
                if (baseValues.groupPriceValue > 0.0)
                {
                    if (n == 0)
                    {
                        emit depthFirstOrder(baseValues.currentPair.symbol, priceDouble, amount, true);
                        groupedPrice = baseValues.groupPriceValue * static_cast<int>(priceDouble / baseValues.groupPriceValue);
                        groupedVolume = amount;
                    }
                    else
                    {
                        bool matchCurrentGroup = priceDouble < groupedPrice + baseValues.groupPriceValue;
                        if (matchCurrentGroup)
                            groupedVolume += amount;
                        if (!matchCurrentGroup || n == asksList.size() - 1)
                        {
                            depthSubmitOrder(baseValues.currentPair.symbol,
                                             &currentAsksMap, groupedPrice + baseValues.groupPriceValue, groupedVolume, true);
                            rowCounter++;
                            groupedVolume = amount;
                            groupedPrice += baseValues.groupPriceValue;
                        }
                    }
                }
                else
                {
                    depthSubmitOrder(baseValues.currentPair.symbol,
                                     &currentAsksMap, priceDouble, amount, true);
                    rowCounter++;
                }
            }
            QList<double> currentAsksList = lastDepthAsksMap.keys();
            for (int n = 0; n < currentAsksList.size(); n++)
                if (qFuzzyIsNull(currentAsksMap.value(currentAsksList.at(n), 0)))
                    depthUpdateOrder(baseValues.currentPair.symbol,
                                     currentAsksList.at(n), 0.0, true);
            lastDepthAsksMap = currentAsksMap;
            groupedPrice = 0.0;
            groupedVolume = 0.0;
            rowCounter = 0;
            for (int n = 1; n < bidsList.size(); n += 2)
            {
                if (baseValues.depthCountLimit && rowCounter >= baseValues.depthCountLimit)
                    break;
                double priceDouble = bidsList.at(n - 1).toString().toDouble();
                double amount      = bidsList.at(n    ).toString().toDouble();
                if (baseValues.groupPriceValue > 0.0)
                {
                    if (n == 0)
                    {
                        emit depthFirstOrder(baseValues.currentPair.symbol, priceDouble, amount, false);
                        groupedPrice = baseValues.groupPriceValue * static_cast<int>(priceDouble / baseValues.groupPriceValue);
                        groupedVolume = amount;
                    }
                    else
                    {
                        bool matchCurrentGroup = priceDouble > groupedPrice - baseValues.groupPriceValue;
                        if (matchCurrentGroup)
                            groupedVolume += amount;
                        if (!matchCurrentGroup || n == bidsList.size() - 1)
                        {
                            depthSubmitOrder(baseValues.currentPair.symbol,
                                             &currentBidsMap, groupedPrice - baseValues.groupPriceValue, groupedVolume, false);
                            rowCounter++;
                            groupedVolume = amount;
                            groupedPrice -= baseValues.groupPriceValue;
                        }
                    }
                }
                else
                {
                    depthSubmitOrder(baseValues.currentPair.symbol,
                                     &currentBidsMap, priceDouble, amount, false);
                    rowCounter++;
                }
            }
            QList<double> currentBidsList = lastDepthBidsMap.keys();
            for (int n = 0; n < currentBidsList.size(); n++)
                if (qFuzzyIsNull(currentBidsMap.value(currentBidsList.at(n), 0)))
                    depthUpdateOrder(baseValues.currentPair.symbol,
                                     currentBidsList.at(n), 0.0, false);
            lastDepthBidsMap = currentBidsMap;
            emit depthSubmitOrders(baseValues.currentPair.symbol, depthAsks, depthBids);
            depthAsks = nullptr;
            depthBids = nullptr;
        }
    }
    break;
    case 202: //info
        {
            QJsonArray account = QJsonDocument::fromJson(data).array();//.object().value("exchange").toArray();
            if (account.isEmpty())
                break;
            QByteArray btcBalance;
            QByteArray usdBalance;
            QJsonArray balances = account.at(0).toObject().value("balances").toArray();
            for (int i = 0; i < balances.size(); ++i)
            {
                if (balances.at(i).toObject().value("currency").toString() == baseValues.currentPair.currAStr)
                    btcBalance = balances.at(i).toObject().value("available").toString().toLatin1();
                if (balances.at(i).toObject().value("currency").toString() == baseValues.currentPair.currBStr)
                    usdBalance = balances.at(i).toObject().value("available").toString().toLatin1();
                if (!btcBalance.isEmpty() && !usdBalance.isEmpty())
                    break;
            }
            if (btcBalance.isEmpty())
                btcBalance = "0";
            if (usdBalance.isEmpty())
                usdBalance = "0";
            if (checkValue(btcBalance, lastBtcBalance))
                emit accBtcBalanceChanged(baseValues.currentPair.symbol, lastBtcBalance);
            if (checkValue(usdBalance, lastUsdBalance))
                emit accUsdBalanceChanged(baseValues.currentPair.symbol, lastUsdBalance);
        }
        break;//info
    case 203: //fee
    {
        QJsonObject fees = QJsonDocument::fromJson(data).object();
        QString makerFee = fees.value("makerRate").toString();
        QString takerFee = fees.value("takerRate").toString();
        if (!makerFee.isEmpty() && !takerFee.isEmpty())
        {
            double fee = qMax(makerFee.toDouble(), takerFee.toDouble()) * 100;
            if (!qFuzzyCompare(fee + 1.0, lastFee + 1.0))
            {
                emit accFeeChanged(baseValues.currentPair.symbol, fee);
                lastFee = fee;
            }
        }
        break;//fee
    }
    case 204://orders
    {
            if (lastOrders != data)
            {
                lastOrders = data;
                auto* orders = new QList<OrderItem>;
                QJsonArray ordersList = QJsonDocument::fromJson(data).array();
                for (int i = 0; i < ordersList.size(); ++i)
                {
                    QJsonObject orderData = ordersList.at(i).toObject();
                    OrderItem currentOrder;
                    currentOrder.date   = orderData.value("createTime").toVariant().toLongLong() / 1000;
                    currentOrder.oid    = orderData.value("id").toString().toLatin1();
                    currentOrder.type   = orderData.value("side").toString() == "SELL";
                    currentOrder.amount = orderData.value("quantity").toString().toDouble();
                    currentOrder.price  = orderData.value("price").toString().toDouble();
                    currentOrder.symbol = IniEngine::getSymbolByRequest(orderData.value("symbol").toString().toLatin1());
                    currentOrder.status = 1;
                    if (currentOrder.isValid())
                        (*orders) << currentOrder;
                }
                if (!orders->empty())
                    emit orderBookChanged(baseValues.currentPair.symbol, orders);
                else
                {
                    delete orders;
                    emit ordersIsEmpty();
                }
            }
            break;//orders
        }
    case 305: //order/cancel
        if (data.startsWith("{\"success\":1,"))
        {
            QByteArray orderId = getMidData("Order #", " canceled", &data);
            emit orderCanceled(baseValues.currentPair.symbol, orderId);
        }
        break;//order/cancel
    case 306:
        if (debugLevel)
            logThread->writeLog("Buy OK: " + data, 2);
        break;//order/buy
    case 307:
        if (debugLevel)
            logThread->writeLog("Sell OK: " + data, 2);
        break;//order/sell
    case 208: //history
        {
            if (data.size() < 50)
                break;
            if (lastHistory != data)
            {
                lastHistory = data;
                QJsonArray historyList = QJsonDocument::fromJson(data).array();
                auto* historyItems = new QList<HistoryItem>;
                for (int n = historyList.size() - 1; n >= 0; --n)
                {
                    QJsonObject logData = historyList.at(n).toObject();
                    qint64 id = logData.value("pageId").toString().toLongLong();
                    if (id <= lastHistoryId)
                        continue;
                    lastHistoryId = id;
                    HistoryItem currentHistoryItem;




 
