//+------------------------------------------------------------------+
//|                                                   EMA Crossover |
//|                                Author: Ayomide Akinbode          |
//+------------------------------------------------------------------+
#property strict

#include <Trade\Trade.mqh>

CTrade trade;

// Input parameters
input int FastEMA = 9;
input int SlowEMA = 21;
input double LotSize = 0.1;
input double StopLoss = 200;     // in points
input double TakeProfit = 400;   // in points

// Handles
int fastHandle, slowHandle;

//+------------------------------------------------------------------+
//| Expert initialization                                           |
//+------------------------------------------------------------------+
int OnInit()
{
   fastHandle = iMA(_Symbol, _Period, FastEMA, 0, MODE_EMA, PRICE_CLOSE);
   slowHandle = iMA(_Symbol, _Period, SlowEMA, 0, MODE_EMA, PRICE_CLOSE);
   
   if(fastHandle == INVALID_HANDLE || slowHandle == INVALID_HANDLE)
   {
      Print("Error creating EMA handles");
      return(INIT_FAILED);
   }
   
   return(INIT_SUCCEEDED);
}

//+------------------------------------------------------------------+
//| Expert deinitialization                                         |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
   if(fastHandle != INVALID_HANDLE) IndicatorRelease(fastHandle);
   if(slowHandle != INVALID_HANDLE) IndicatorRelease(slowHandle);
}

//+------------------------------------------------------------------+
//| Expert tick                                                     |
//+------------------------------------------------------------------+
void OnTick()
{
   double fastEMA[], slowEMA[];
   
   if(CopyBuffer(fastHandle, 0, 0, 3, fastEMA) < 0 ||
      CopyBuffer(slowHandle, 0, 0, 3, slowEMA) < 0)
      return;
   
   double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
   double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
   
   bool hasPosition = false;
   ENUM_POSITION_TYPE posType;
   
   for(int i = PositionsTotal() - 1; i >= 0; i--)
   {
      ulong ticket = PositionGetTicket(i);
      if(PositionSelectByTicket(ticket))
      {
         if(PositionGetString(POSITION_SYMBOL) == _Symbol)
         {
            hasPosition = true;
            posType = (ENUM_POSITION_TYPE)PositionGetInteger(POSITION_TYPE);
         }
      }
   }
   
   // Buy signal
   if(fastEMA[1] < slowEMA[1] && fastEMA[0] > slowEMA[0] && !hasPosition)
   {
      trade.Buy(LotSize, _Symbol, ask, 
                ask - StopLoss * _Point, 
                ask + TakeProfit * _Point);
   }
   
   // Sell signal
   if(fastEMA[1] > slowEMA[1] && fastEMA[0] < slowEMA[0] && !hasPosition)
   {
      trade.Sell(LotSize, _Symbol, bid, 
                 bid + StopLoss * _Point, 
                 bid - TakeProfit * _Point);
   }
   
   // Close opposite position if crossover happens
   if(hasPosition)
   {
      if(posType == POSITION_TYPE_BUY && fastEMA[0] < slowEMA[0])
         trade.PositionClose(_Symbol);
      else if(posType == POSITION_TYPE_SELL && fastEMA[0] > slowEMA[0])
         trade.PositionClose(_Symbol);
   }
}
