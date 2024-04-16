# myhub
存放常使用工具
void BMS_SendCanPackage(uint8_t cmd)
{
	 J1939_id_t idObj;
	 idObj.SourceAdd = 0xF4;
	 idObj.PduSpecial =  0x50;
	 idObj.Byte3.Priority = 7;
	 idObj.Byte3.DP = 0;
	 idObj.Byte3.R = 0;
	
	 if(cmd == 0x06)
	 {
		idObj.PduFormat = cmd;
    		          
        #ifdef USE_MCU_GET_TEMP
        CanPack06Obj.temperature = AdcValueObj.tempBAT;
        #else
        CanPack06Obj.temperature = Afe309.TemperK[1];
        #endif
        CanPack06Obj.soc         = SystemStatusObj.soc;
         
        uint8_t soh = 0;
        if(SystemStatusObj.BQSetupStep != 4) //BQ error or Pack Voltage too low, SOH = 0。
        {
            if(fault.SYS.byte == 0) //bms ok
            {
                soh = MID_GetSOH();
            }
            else
            {
                soh = fault.SYS.byte;
            }
        }
        CanPack06Obj.soh         = soh;
        CanPack06Obj.total_volt  = MID_GetTotalCellVoltage(); //AdcValueObj.voltagePACK;
        
        int avgCurrent = SystemStatusObj.RealtimeCurrent;
        if(SystemStatusObj.flagTestMode != 1) // not test mode
        {
            avgCurrent = MID_GetAverageCurrent();
            if(abs(avgCurrent) < 50) avgCurrent = 0;
        }
        CanPack06Obj.ave_current = avgCurrent/100; //单位0.1A
         
        CanPack06Obj.hw_ver      = HARDWARE_VERSION;
        CanPack06Obj.sf_ver      = FIRMWARE_VERSION;
         
        uint8_t DsgMosStat = GPIO_BMS_GetFeedback_DischargeMOS();
        uint8_t ChgMosStat = GPIO_BMS_GetFeedback_ChargeMOS();
        uint8_t UnderVolDsgProtect = fault.STAT.bit.UV;
        uint8_t gpsStat = GPIO_Get_GPS_Stat();
        uint8_t maintainStat = RomParamObj.MaintainStatus;
        CanPack06Obj.opt_bit_ste =  ((ChgMosStat||DsgMosStat)?1:0 << 0) | (UnderVolDsgProtect << 1) | (gpsStat << 2) | (maintainStat << 3);

		CanPack06Obj.status[0] = fault.STAT.bit.OV | (fault.MCU.bit.UV << 1) | (fault.MCU.bit.OCD << 2) | (fault.MCU.bit.OTD <<4) | (fault.MCU.bit.OTC << 5) | (fault.MCU.bit.UTD << 6) | (fault.MCU.bit.UTC << 7);
        CanPack06Obj.status[1]  = SOFT_OCD/1000;
        CanPack06Obj.status[2]  = (OVER_CHARGE_CURRENT/1000 & 0x7F) | ((SystemStatusObj.RealtimeCurrent > 100? 1:0) << 7);
        
        /* Ascii 0x31=48V,0x32=60V，0x33=72V */
        if((PACK_CELL_NUMBER == 20)
         ||((PACK_CELL_NUMBER == 17) && (CELL_TYPE == 2)))    //铁锂20串或三元17串电芯，60V
        {
            CanPack06Obj.id_code[0] = 0x32;
        }
        else if((PACK_CELL_NUMBER == 20) && (CELL_TYPE == 3)) //固态电芯，20串，72V 
        {
            CanPack06Obj.id_code[0] = 0x33;
        }
        else if((PACK_CELL_NUMBER == 24) && (CELL_TYPE == 1)) //铁锂电芯，24串，72V
        {
            CanPack06Obj.id_code[0] = 0x33;
        }
        else if( PACK_CELL_NUMBER == 16)                      //铁锂电芯，16串，48V
        {
            CanPack06Obj.id_code[0] = 0x31;
        }
        else if((PACK_CELL_NUMBER == 13) && (CELL_TYPE == 3)) //固态电芯，13串，48V 
        {
            CanPack06Obj.id_code[0] = 0x31;
        }
        else if((PACK_CELL_NUMBER == 14) && (CELL_TYPE == 2)) //三元电芯，14串，48V 
        {
            CanPack06Obj.id_code[0] = 0x31;
        }
        else if((PACK_CELL_NUMBER == 13) && (CELL_TYPE == 2)) //孚能三元电芯，13串，48V 
        {
            CanPack06Obj.id_code[0] = 0x31;
        }
        else
        {
            CanPack06Obj.id_code[0] = 0x30;
        }
        
        for(uint8_t i = 0; i < 15; i++)
        {
            CanPack06Obj.id_code[i+1] = RomParamObj.PackId[i+1];
        }
				
        uint16_t crcSum = CalcCRC16Sum((uint8_t*)&CanPack06Obj,sizeof(CanPack06Cmd_t)-2);
        CanPack06Obj.crc_sum = (crcSum<<8)|(crcSum>>8);
				
        send_bms_data_packet(idObj, (uint8_t*)&CanPack06Obj, sizeof(CanPack06Cmd_t));				
	 }
	 else if(cmd == 0x07)
	 {
         idObj.PduFormat = cmd;

         CanPack07Obj.residue_capacity  = RomParamObj.LastCap_mAS/3600;
         CanPack07Obj.full_capacity     = DESIGN_CAP_mAH;
         CanPack07Obj.cycle_charge_cnt  = RomParamObj.CycleTimes;
         uint8_t i;
         for(i = 0; i < 6; i++)
         {
            CanPack07Obj.manufacturer_name[i] = PACKAGE_INFO[i];
         }
         
         for(i = 0; i < 20; i++)
         {
            CanPack07Obj.cell_volt[i] = SystemStatusObj.CellVoltage[i];
         }
         
         uint16_t crcSum = CalcCRC16Sum((uint8_t*)&CanPack07Obj,sizeof(CanPack07Cmd_t)-2);
         CanPack07Obj.crc_sum = (crcSum<<8)|(crcSum>>8);

         send_bms_data_packet(idObj, (uint8_t*)&CanPack07Obj, sizeof(CanPack07Cmd_t));
	 }
}



typedef struct tagLion_DataStruct
{       
    uint16_t temperature;
    uint8_t  soc;
    uint8_t  soh;
    uint32_t total_volt;
    uint16_t ave_current;
    uint8_t hw_ver;
    uint8_t sf_ver;
    uint8_t opt_bit_ste;
    uint8_t status[3];
    uint8_t id_code[16];
    uint16_t  crc_sum;
}CanPack06Cmd_t;
CanPack06Cmd_t CanPack06Obj;

typedef struct
{       
    uint16_t residue_capacity;            
    uint16_t full_capacity;               
    uint16_t cycle_charge_cnt;            
    uint8_t manufacturer_name[6];         
    uint16_t cell_volt[20];
    uint16_t  crc_sum;
}CanPack07Cmd_t;
CanPack07Cmd_t CanPack07Obj;


static unsigned short CalcCRC16Sum(const void *s, int n) 
{ 
	 unsigned short c = 0xffff; 
	 for(int k = 0; k < n; k++) 
	 { 
		 unsigned short b=(((unsigned char *)s)[k]); 
		 for(char i=0; i<8; i++) 
		 { 
			 c = ((b^c)&1) ? (c>>1)^0xA001 : (c>>1); 
			 b >>= 1;
		 } 
	 }
	 return (c << 8)|(c >> 8);
}
