1 删除多余的文件
2 在arch_mian.c 中的384行 增加以下代码		
    GLOBAL_INT_START();
		uart0.param.baud_rate = UART_BAUDRATE_9600;
    uart0.param.rx_pin_no = 13;
    uart0.param.tx_pin_no = 12;
    uart0.param.tx_dma = 0;
    uart0.param.rx_dma = 0;
    app_uart_init(&uart0.inst);
		printf("start\n");
    
3 在 app_uart.c中的430行 增加以下代码

unsigned char UartPutc(unsigned char my_ch)
{
    app_uart_inst_t uart0 = UART_INSTANCE(0);
    app_uart_inst_t *inst = CONTAINER_OF(&uart0.inst, app_uart_inst_t, inst);
		uart_universal_func.sys_stat_func(inst,UART_READ_START);
    reg_uart_t *regss = inst->reg;
	
    FIELD_WR(regss,DLH_IER,UART_ERBFI,Received_Data_Available_Interrupt_Enabled);
    while((FIELD_RD(regss,USR,UART_TFNF)==Transmit_FIFO_Full));
    regss->RBR_THR_DLL = my_ch;
    return (my_ch);
}

int fputc(int ch, FILE *f)   //重定义 fputc 函数
{
  return (UartPutc(ch));
}
以上可以实现在工程中调用 printf()函数打印数据

程序开始
初始化完成之后，协议栈回报一个 GAPM_DEVICE_READY_IND 信号量 触发对应的句柄 
在句柄函数中 发送 GAPM_RESET_CMD命令给GAPM 
完成之后，GAPM将触发通用事件 GAPM_CMP_EVT 
其中stwith()语句找是什么命令触发的通用事件
在对应的通用事件中 发送 GAPM_SET_DEV_CONFIG_CMD 命令给GAPM
同样的 完成之后继续操作相关的命令


增加OTA功能，需要增加在	  
static void osapp_gapm_profile_added_ind_handler(ke_msg_id_t const msgid, struct gapm_profile_added_ind const *param,
                                                  ke_task_id_t const dest_id,ke_task_id_t const src_id)
{
  /*在osapp_gapm_profile_added_ind_handler函数中的参数里，将更改void const *param 成struct gapm_profile_added_ind const *param*/
	/*增添以下代码*/
	  if(param->prf_task_id == TASK_ID_DISC){
        osapp_add_bxotas_task();
    }
    else if(param->prf_task_id==TASK_ID_BXOTAS){
        osapp_bxotas_config();
    }
}
}
//检查OTA设备是否开始的状态
static void osapp_bxotas_start_cfm(uint8_t status)
{
    struct bxotas_start_cfm *cfm = AHI_MSG_ALLOC(BXOTAS_START_CFM, TASK_ID_BXOTAS, bxotas_start_cfm);
    cfm->status = status;
    osapp_ahi_msg_send(cfm, sizeof(struct bxotas_start_cfm), portMAX_DELAY);
}

static void osapp_bxotas_start_req_ind_handler(ke_msg_id_t const msgid,struct bxotas_start_req_ind const *param,
ke_task_id_t const dest_id,ke_task_id_t const src_id)
{
    printf("OTA START:%d,%d\n",param->conidx,param->segment_data_max_length);
    osapp_bxotas_start_cfm(OTA_REQ_ACCEPTED);
}
//OTA完成之后，延时一段时间，设备重启
static void osapp_bxotas_finish_ind_handler(ke_msg_id_t const msgid,void const *param,ke_task_id_t const dest_id,
ke_task_id_t const src_id)
{
    printf("OTA_DONE\n");
    /* do not reset chip immediately */
    BX_DELAY_US(1000000);
    platform_reset(0);
}
并添加相关OTA文件的头文件路径以及在对应的c文件中包含对应的头文件

扫描
static int8_t app_correct_rssi(int8_t rssi)
{
    int8_t accurate_rssi;   
    if(rssi <= -17)
    {
        // 1.026 * rssi + 8.641 + 0.5;
        accurate_rssi = rssi + 9;
    }
    else if(rssi <= -4)
    {
        accurate_rssi = rssi + 11;
    }
    else
    {
        accurate_rssi = rssi + 6;
    }
    return accurate_rssi;
}
/**这里需要改动过滤重复扫描到广播设备mac的操作**/
static void osapp_gapm_adv_report_ind_handler(ke_msg_id_t const msgid, void const *param,ke_task_id_t const dest_id,
                                              ke_task_id_t const src_id)
{

    struct adv_report const *report = param;
		uint8_t count=0;
		uint8_t index = report->data_len;
		bool found = false;
    const int8_t rssi = app_correct_rssi(report->rssi) ;   
    for (uint8_t i = 0; i < adv_data.app_env_inq_idx; i++) //adv_addr_type == 1 是Random adv_addr_type == 0  Public
    {
        if (0 == memcmp(&adv_data.inq_addr[i], &report->adv_addr,6))
        {
            found = true;
            break;
        }
    }
		if(!found)  
		{
			  memset(&adv_data.inq_addr[adv_data.app_env_inq_idx],0,6);
				adv_data.adv_addr_type[adv_data.app_env_inq_idx] = report->adv_addr_type;
				memcpy(&adv_data.inq_addr[adv_data.app_env_inq_idx],&report->adv_addr,6);
				printf("%d.%s  %02X:%02X:%02X:%02X:%02X:%02X   ",
									adv_data.app_env_inq_idx,
									report->adv_addr_type ? "Random" : "Public",
									adv_data.inq_addr[adv_data.app_env_inq_idx].addr[5],
									adv_data.inq_addr[adv_data.app_env_inq_idx].addr[4],
									adv_data.inq_addr[adv_data.app_env_inq_idx].addr[3],
									adv_data.inq_addr[adv_data.app_env_inq_idx].addr[2],
									adv_data.inq_addr[adv_data.app_env_inq_idx].addr[1],
									adv_data.inq_addr[adv_data.app_env_inq_idx].addr[0]);
			adv_data.app_env_inq_idx++;
		  printf("RSSI = %dbm",rssi);
			printf(" \t");

			if(report->data[4] == 8||report->data[4] == 9 )//判断广播设备是否广播设备的名称（完整的名称与短）
			{
				printf("%s",&report->data[5]); 
			}
			printf("\n",report->data);
		}
		
}

