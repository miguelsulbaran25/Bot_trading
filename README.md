# Bot_trading
Bot de trading en Mql5
#property copyright "Copyright 2024, MetaQuotes Ltd."
#property link      "https://www.mql5.com"
#property version   "1.00"
//+------------------------------------------------------------------+
//| include                                                          |
//+------------------------------------------------------------------+
#include <Trade\Trade.mqh>

//+------------------------------------------------------------------+
//| Inputs                                                           |
//+------------------------------------------------------------------+

input group "==== general ====";
static input long InpMagicnumber = 564892; // numero magico
static input double InpLotSize = 0.01; // lotaje de perdida
input group  "==== general ====";
input int    InpStoppLoss = 200; // stop loss = off
input int    InpTakeProfit = 0; // take profit
input bool   InpCloseSignal = false; // cerrar trade de la posicion opuesta
input group "====Estocastico====";
input int         InpKPeriod    = 21; // k periodo
input int         InpUPPerLevel    = 80; // nivel supior 
//+------------------------------------------------------------------+
//| variables globales                                               |
//+------------------------------------------------------------------+

int handle;
double buffermain[];
MqlTick cT;
CTrade trade;

//+------------------------------------------------------------------+
//| Expert advisor                                                   |
//+------------------------------------------------------------------+
   int OnInit()
{
   // verificar entradas de usuario
   if(!CheckInputs()){return INIT_PARAMETERS_INCORRECT;}
   
   // numero magico para el objeto comercial
   
   trade.SetExpertMagicNumber(InpMagicnumber);
   
   // crear el indicador de rango
   handle = iStochastic(_Symbol,PERIOD_CURRENT,InpKPeriod,1,3,MODE_SMA,STO_LOWHIGH);
   
   if(handle==INVALID_HANDLE)
   {
      Alert("no se pudo crear el identificador del indicador");
      return INIT_FAILED;
   }
   
   // establecer buffer como serie
   
   ArraySetAsSeries(buffermain,true);
   
    return(INIT_SUCCEEDED);
}

//+------------------------------------------------------------------+
//| Expert deinitialization function                                 |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
  {
   // mango liberacion del indicador
   
   if(handle!=INVALID_HANDLE)
   {
    IndicatorRelease(handle);
   }
  }
//+------------------------------------------------------------------+
//| Expert tick function                                             |
//+------------------------------------------------------------------+
void OnTick()
  {
   // verifiquemos sobre open tick
   if(!IsNewBar()){return;}
   
   //nose pudo obtener el tickeck
   if(!SymbolInfoTick(_Symbol,cT)){printf("no se pudo obtener el símbolo"); return;}
   
   // obtener valores del indicador
   if(CopyBuffer(handle,0,1,2,buffermain)!=2)
   {
      printf("no pudimos obtener lo valores indicado");
      return;
   }
   
   //contemos las posiciones abiertas
   
   int cntBuy, cntSell;
   
   if(!CountOpenPositions(cntBuy,cntSell)){
      printf("no hay posiciones abiertas");
      return;
      }
      
      //verifique posicion compra
      if(cntBuy==0 && buffermain[0] <=(100-InpUPPerLevel) && buffermain [1] > (100-InpUPPerLevel))
      {
         
         if(InpCloseSignal){if(!ClosePositions(2)){return;}}
         double sl = InpStoppLoss==0 ? 0 : cT.bid - InpStoppLoss * _Point;
         double tp = InpStoppLoss==0 ? 0 : cT.bid + InpTakeProfit * _Point;
         if(!NormalizePrice(sl)){return;}
         if(!NormalizePrice(tp)){return;}
         trade.PositionOpen(_Symbol,ORDER_TYPE_BUY,InpLotSize,cT.ask,sl,tp,"Ea estocastico");
      }
       //verifique posicion venta
      if(cntSell==0 && buffermain[0] >= InpUPPerLevel && buffermain [1] < InpUPPerLevel)
      {
         
         if(InpCloseSignal){if(!ClosePositions(1)){return;}}
         double sl = InpStoppLoss==0 ? 0 : cT.ask + InpStoppLoss * _Point;
         double tp = InpStoppLoss==0 ? 0 : cT.ask - InpTakeProfit * _Point;
         if(!NormalizePrice(sl)){return;}
         if(!NormalizePrice(tp)){return;}
         trade.PositionOpen(_Symbol,ORDER_TYPE_SELL,InpLotSize,cT.bid,sl,tp,"Ea estocastico");
      }
  }

//+------------------------------------------------------------------+
//| funciones personalizadas                                         |
//+------------------------------------------------------------------+

// verifica la entrada del usuario
bool CheckInputs(){
   
   if(InpMagicnumber<=0){
   Alert("número mágico ingresado incorrectamente <=0");
   return false;
   }
   
   if(InpLotSize<=0 || InpLotSize>10){
   Alert("tamaño de lote <=0 o >10");
   return false;
   }
   
   if(InpStoppLoss<0){
   Alert("stop loss incorecto <0");
   return false;
   }
   
   if(InpTakeProfit<0){
   Alert("take profit incorrecto <0");
   return false;
   }
   //
   if(!InpCloseSignal && InpStoppLoss==0){
   Alert("la señal de cierre como falsa y sin stop loss ");
   return false;
   }
   if(InpKPeriod<=0){
   Alert("numero de periodo ingresado incorrectamente <=0");
   return false;
   }
   if(InpUPPerLevel<=50 || InpUPPerLevel>100){
   Alert("el numero ingresado superior <=50 o >=100");
   return false;
   }
   
   return true;
}

// Comprueba si tenemos una entrada abierta

bool IsNewBar(){

   static datetime previusTime = 0;
   datetime currenTime = iTime(_Symbol,PERIOD_CURRENT,0);
   if(previusTime!=currenTime){
   previusTime=currenTime;
   return true;
   }
   return false;

}

// contar posicion abierta
bool CountOpenPositions(int &cntBuy, int &cntSell){
   cntBuy= 0;
   cntSell = 0;
   int total = PositionsTotal();
   for(int i=total-1; i>=0; i--){
      ulong ticket = PositionGetTicket(i);
      if(ticket<=0){Print("No se pudo conseguir la posicion"); return false;}
      if(!PositionSelectByTicket(ticket)){Print("no se pudo seleccionar la posición"); return false;}
      long magic;
      if(!PositionGetInteger(POSITION_MAGIC,magic)){Print("No se pudo obtener la posición del número mágico"); return false;}
      if(magic==InpMagicnumber){
         long type;
         if(!PositionGetInteger(POSITION_TYPE,type)){Print("no se pudo obtener el tipo de posición"); return false;}
         if(type==POSITION_TYPE_BUY){cntBuy++;}
         if(type==POSITION_TYPE_SELL){cntSell++;}
   }
   
   
   }
   
   return true;
}

// normalizar precio

bool NormalizePrice(double &price){

   double tickSize=0;
   if(!SymbolInfoDouble(_Symbol,SYMBOL_TRADE_TICK_SIZE,tickSize)){
   Print("no se pudo obtener el tamaño del tikect");
   return false;
   }   
   price = NormalizeDouble(MathRound(price/tickSize)*tickSize,_Digits);
   
   return true;
}


// cerrrar prositions
bool ClosePositions(int all_buy_sell){

 int total = PositionsTotal();
 for(int i=total-1; i>=0; i--){
   ulong ticket = PositionGetTicket(i);
   if(ticket<=0){printf("no se pudo obtener la posicion del ticket"); return false;}
   if(!PositionSelectByTicket(ticket)){Print("no se pudo obtener la posicion"); return false;}
   long magic;
   if(!PositionGetInteger(POSITION_MAGIC,magic)){Print("No se pudo obtener la posición del número mágico."); return false;}
   if(magic==InpMagicnumber){
      long type;
      if(!PositionGetInteger(POSITION_TYPE,type)){printf("no se pudo obtener el tipo de posición"); return false;}
      if(all_buy_sell==1 && type==POSITION_TYPE_SELL){continue;}
      if(all_buy_sell==2 && type==POSITION_TYPE_BUY){continue;}
      trade.PositionClose(ticket);
      if(trade.ResultRetcode()!=TRADE_RETCODE_DONE){
         Print("no se pudo cerrar la position del ticket:",
               (string)ticket,"resultado:",(string)trade.ResultRetcode(),":",trade.CheckResultRetcodeDescription());
      }
   }
 }
 
  return true;
}
