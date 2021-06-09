# FXOSMouseHandler
Description for Mouse implementation in WDC C



## Interrupt Handler in ASM calling C IRQHandler
```
IRQ:	
		setal 	

		phb		; save Data Bank
		phd		; save Direct Page Register
		pha
		phx
		phy

		setas

		setal
		XREF  ~~IRQHandler
		jsl   ~~IRQHandler

		setal
		
		ply
		plx
		pla
		pld		; restore Direct Page Reg
		plb		; Restore Data Bank
		
		RTI
```

## C IRQHandler to begin interrupt detection process

```
void IRQHandler(void)
{
	IRQDATA data;
		
	if(INT_PENDING_REG0[0]!=0)
	{
		k_dispatch_reg0(&data);
	}
	
	if(INT_PENDING_REG1[0]!=0)
	{
		k_dispatch_reg1(&data);	
	}
	
	if(INT_PENDING_REG2[0]!=0)
	{
		k_dispatch_reg2(&data);
	}

	if(INT_PENDING_REG3[0]!=0)
	{
		k_dispatch_reg3(&data);
	}

	return;
}
```

## Dispatch 0, Interrupt 7 (mouse) Handler

```
void k_dispatch_reg0(PIRQDATA pIRQx)
{
...
	if(INT_PENDING_REG0[0] & FNX0_INT07_MOUSE)
	{
		_irq_keyboardTimeout = 0;
		while((STATUS_PORT[0] & 0x01) && spc < 4)
		{
			mptr = MOUSE_PTR[0];
			kbd  = KBD_INPT_BUF[0];

			MOUSE_PTR_BYTE0[mptr]   = kbd;			
			FXOS_MOUSE_BYTE[mptr]   = kbd;
				
			mptr++;	
			spc++;

			if(mptr >  2)
			{
				MOUSE_PTR[0] = 0;
				
				FXOS_MOUSE_BYTE_T   = MOUSE_PTR_BYTE0[0];
				FXOS_MOUSE_BYTE_X_L = MOUSE_PTR_X_POS_L[0];
				FXOS_MOUSE_BYTE_X_H = MOUSE_PTR_X_POS_H[0];
				FXOS_MOUSE_BYTE_Y_L = MOUSE_PTR_Y_POS_L[0];
				FXOS_MOUSE_BYTE_Y_H = MOUSE_PTR_Y_POS_H[0];

				k_irq_device_event(IRQE_MOUSE,_pseudo_timer,&FXOS_MOUSE_BYTE_T);

				spc  = 0;
				mptr = 0;
			}
			else
			{
				MOUSE_PTR[0] = mptr;
			}

			spc++;
		}
		INT_PENDING_REG0[0] &= FNX0_INT07_MOUSE;
	}
  ...
}

```

## Create Window Manager Event from IRQ

```



void k_irq_device_event(MSGIRQ type,ULONG timer,void FAR *data)
{
	BOOL bRet = FALSE;

	PFXOSMESSAGE pmsg = k_create_msg(type,timer,data);
	if(pmsg)
	{
		if(type == IRQE_COM1)
		{
			k_mem_deallocate_heap(pmsg);
			return;
		}

		if(type == IRQE_CTLR_RESET)
		{
			if(timer == -1)
			{
				_k_mouseState.buttonLeftDown   = FALSE;
				_k_mouseState.buttonMiddleDown = FALSE;
				_k_mouseState.buttonRightDown  = FALSE;
				_k_mouseState.lastEvent = 0;
			}
		}

		if(type == IRQE_MOUSE)
			pmsg = k_updateMouseState(pmsg,timer,data);

		bRet = k_enqueue(_k_eventQueue,pmsg);
		if(!bRet)
		{
			k_debug_integer("k_irq_device_event:fail:type:",type);
		}
	}
}

PFXOSMESSAGE k_create_msg(MSGIRQ type,ULONG timer,void FAR *data)
{
	PFXOSMESSAGE pmsg = (PFXOSMESSAGE)k_mem_allocate_heap(sizeof(FXOSMESSAGE));
	if(pmsg)
	{
		memset(pmsg,0,sizeof(FXOSMESSAGE));
		pmsg->pheap = (LPVOID)0xFFFFFF;
		pmsg->dest = FX_MSG_DEFAULT;
		pmsg->src  = FX_MSG_DEFAULT;
		//k_debug_integer("k_create_msg:",type);
		switch(type)
		{
...
		case IRQE_MOUSE:
			pmsg->type = FX_MOUSE_MOVE;
			pmsg->data[0] = *((BYTE*)data) & 7; // mouse byte 1
			pmsg->data[1] = ((LPCHAR)data)[1];
			pmsg->data[2] = ((LPCHAR)data)[2];
			pmsg->data[3] = ((LPCHAR)data)[3];
			pmsg->data[4] = ((LPCHAR)data)[4];

			break;
...
		default:
			pmsg->type = 99;//IRQE_UNKNOWN
			break;
		}

	}
	return pmsg;
}

```

## Detect and Create Various Mouse States based on time/button/movement 

```
PFXOSMESSAGE k_updateMouseState(PFXOSMESSAGE pmsg,ULONG timer,void FAR *data)
{
	ULONG lastTimer = _k_mouseState.lastEvent;

	INT statusLeft   = ((LPCHAR)data)[0] & 1;
	INT statusRight  = ((LPCHAR)data)[0] & 2;
	INT statusMiddle = ((LPCHAR)data)[0] & 4;

	if(_k_mouseState.buttonLeftDown)
	{
		if(statusLeft)
		{
			_k_mouseState.buttonLeftDown = 1;
			if((timer - _k_mouseState.lastLeftDown) > 5)
			{
				//k_debug_string("k_updateMouseState:LeftMouseDRAG\r\n");
				pmsg->type = FX_LBUTTON_DRAG;
			}
		}
		else
		{
			//k_debug_string("k_updateMouseState:LeftMouseUp\r\n");
			pmsg->type = FX_LBUTTON_UP;
			_k_mouseState.buttonLeftDown = 0;
		}
	}
	else
	{
		if(statusLeft)
		{
			_k_mouseState.buttonLeftDown = 1;

			if((timer - _k_mouseState.lastLeftDown) < 5)
			{
				//k_debug_integer("k_updateMouseState:LeftMouseDblClick:",timer - _k_mouseState.lastLeftDown);
				pmsg->type = FX_LBUTTON_DBLCLICK;
			}
			else
			{
				//k_debug_string("k_updateMouseState:LeftMouseDown\r\n");
				pmsg->type = FX_LBUTTON_DOWN;
			}
			_k_mouseState.lastLeftDown = timer;
		}
		else
		{

			_k_mouseState.buttonLeftDown = 0;
		}
	}

	if(_k_mouseState.buttonRightDown)
	{
		if(statusRight)
		{
			_k_mouseState.buttonRightDown = 1;
			if((timer - _k_mouseState.lastRightDown) > 5)
			{
				//k_debug_string("k_updateMouseState:LeftMouseDRAG\r\n");
				pmsg->type = FX_RBUTTON_DRAG;
			}
		}
		else
		{
			//k_debug_string("k_updateMouseState:RightMouseUp\r\n");
			pmsg->type = FX_RBUTTON_UP;
			_k_mouseState.buttonRightDown = 0;
		}
	}
	else
	{
		if(statusRight)
		{
			_k_mouseState.buttonRightDown = 1;

			if((timer - _k_mouseState.lastRightDown) < 5)
			{
				//k_debug_integer("k_updateMouseState:RightMouseDblClick:",timer - _k_mouseState.lastRightDown);
				pmsg->type = FX_RBUTTON_DBLCLICK;
			}
			else
			{
				//k_debug_string("k_updateMouseState:RightMouseDown\r\n");
				pmsg->type = FX_RBUTTON_DOWN;
			}
			_k_mouseState.lastRightDown = timer;
		}
		else
		{

			_k_mouseState.buttonRightDown = 0;
		}
	}

	if(_k_mouseState.buttonMiddleDown)
	{
		if(statusMiddle)
		{
			_k_mouseState.buttonMiddleDown = 1;
			if((timer - _k_mouseState.lastMiddleDown) > 5)
			{
				//k_debug_string("k_updateMouseState:MiddletMouseDRAG\r\n");
				pmsg->type = FX_MBUTTON_DRAG;
			}
		}
		else
		{
			//k_debug_string("k_updateMouseState:MiddleMouseUp\r\n");
			pmsg->type = FX_MBUTTON_UP;
			_k_mouseState.buttonMiddleDown = 0;
		}
	}
	else
	{
		if(statusMiddle)
		{
			_k_mouseState.buttonMiddleDown = 1;

			if((timer - _k_mouseState.lastMiddleDown) < 5)
			{
				//k_debug_integer("k_updateMouseState:MiddleMouseDblClick:",timer - _k_mouseState.lastMiddleDown);
				pmsg->type = FX_MBUTTON_DBLCLICK;
			}
			else
			{
				//k_debug_string("k_updateMouseState:MiddleMouseDown\r\n");
				pmsg->type = FX_MBUTTON_DOWN;
			}
			_k_mouseState.lastMiddleDown = timer;
		}
		else
		{

			_k_mouseState.buttonMiddleDown = 0;
		}
	}

	_k_mouseState.lastEvent = timer;

	return pmsg;
}

```
