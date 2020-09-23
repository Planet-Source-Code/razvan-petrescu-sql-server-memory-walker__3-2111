<div align="center">

## SQL server memory walker


</div>

### Description

Extended stored procedure that shows RAM allocation of a SQL Server. Look at the code for an explanation of the output.
 
### More Info
 
Number of regions to scan (use: xp_mwalk no_of_regions ).

Compile the code using VC's extended stored procedure settings (with opends60.lib)

Each region's protection and size.


<span>             |<span>
---                |---
**Submitted On**   |
**By**             |[Razvan Petrescu](https://github.com/Planet-Source-Code/PSCIndex/blob/master/ByAuthor/razvan-petrescu.md)
**Level**          |Intermediate
**User Rating**    |5.0 (10 globes from 2 users)
**Compatibility**  |C, C\+\+ \(general\), Microsoft Visual C\+\+
**Category**       |[Databases](https://github.com/Planet-Source-Code/PSCIndex/blob/master/ByCategory/databases__3-5.md)
**World**          |[C / C\+\+](https://github.com/Planet-Source-Code/PSCIndex/blob/master/ByWorld/c-c.md)
**Archive File**   |[](https://github.com/Planet-Source-Code/razvan-petrescu-sql-server-memory-walker__3-2111/archive/master.zip)

### API Declarations

srv.h


### Source Code

```
/*
 * SQL Server memory walker extended stored procedure
 *
 * rp 08/03/2001
 *
 * Look at the code to understand the output.
 *
 */
#pragma once
#define WIN32_LEAN_AND_MEAN
#include <windows.h>
#include <Srv.h>
#define XP_NOERROR    0
#define XP_ERROR    1
#define MAXCOLNAME				25
#define MAXNAME					25
#define MAXTEXT					255
#define SRV_MAXERROR			20000
#define XP_ERROR_PARMS			SRV_MAXERROR + 1
#define _E_PAR					1
#define _E_PDTYPE				2
#define _E_PRANGE				3
#define _ERR( szErrMsg, e ){ \
	srv_sendmsg( srvproc, SRV_MSG_ERROR, XP_ERROR_PARMS + e, SRV_INFO, (DBTINYINT)0, NULL, 0, 0, szErrMsg, SRV_NULLTERM ); \
	srv_senddone( srvproc, SRV_DONE_FINAL, 0, 0 ); \
	return XP_ERROR; }
#ifdef __cplusplus
extern "C" {
#endif
RETCODE __declspec(dllexport) xp_mwalk(SRV_PROC *srvproc);
#ifdef __cplusplus
}
#endif
BOOL APIENTRY DllMain( HANDLE hModule,
      DWORD ul_reason_for_call,
      LPVOID lpReserved
					 )
{
 return TRUE;
}
/*
 * xp_mwalk implementation
 */
RETCODE __declspec(dllexport) xp_mwalk(SRV_PROC *srvproc)
{
	DBCHAR						szMessage[ MAXTEXT ];
	int							iHops, iHops_c;
	int							iPType, iPLen;
	MEMORY_BASIC_INFORMATION	mbi;
	SYSTEM_INFO					si;
	PBYTE						lpBase, lpAdd;
	DWORD						dwAdd;
	DBCHAR						szState[5] = { '\0' };
	DBCHAR						szProtect[30] = { '\0' };
	DBCHAR						szType[4] = { '\0' };
	int iParamCount = srv_rpcparams( srvproc );
	if( iParamCount != 1 ) //can't test for 0
	{
		wsprintf( szMessage, "Use: xp_mwalk no_of_iterations (%u)", iParamCount );
		_ERR( szMessage, _E_PAR )
	}
	iPLen = srv_paramlen( srvproc, 1 );
	iPType = srv_paramtype( srvproc, 1 );
	if( iPLen <= 0 )
		_ERR( "Parameter can't be NULL.", _E_PDTYPE )
	if( iPType != SRVINTN )
		_ERR( "Parameter must be integer.", _E_PDTYPE )
	iHops = iHops_c = (int) *((int*)srv_paramdata( srvproc, 1 ));
	if( iHops < 1 )
		_ERR( "Parameter must be >= 1.", _E_PRANGE )
	GetSystemInfo( &si );
	wsprintf( szMessage, "Platform info: \x0d\x0a"
		"Page size [%u] Granularity [%u] No.processors "
		"[%u] Min.addr.[0x%x] Max.addr.[0x%x]",
		si.dwPageSize, si.dwAllocationGranularity, si.dwNumberOfProcessors,
		si.lpMinimumApplicationAddress,	si.lpMaximumApplicationAddress );
	srv_sendmsg( srvproc, SRV_MSG_INFO, XP_NOERROR, SRV_INFO, (DBTINYINT)0,
		NULL, 0, 0, szMessage, SRV_NULLTERM );
	srv_senddone( srvproc, SRV_DONE_MORE, 0, 0 );
	lpBase = (PBYTE)(si.lpMinimumApplicationAddress);
	lpAdd = lpBase;
	do
	{
		dwAdd = VirtualQuery((LPCVOID)lpAdd, &mbi, sizeof( mbi ));
		if( mbi.State & MEM_COMMIT )
			strcat( szState, "C" );
		if( mbi.State & MEM_RESERVE )
			strcat( szState, "R" );
		if( mbi.State & MEM_FREE )
			strcat( szState, "F" );
		if( mbi.Protect & PAGE_READONLY )
			strcat( szProtect, "Ro" );
		if( mbi.Protect & PAGE_READWRITE )
			strcat( szProtect, "Rw" );
		if( mbi.Protect & PAGE_WRITECOPY )
			strcat( szProtect, "Wc" );
		if( mbi.Protect & PAGE_EXECUTE )
			strcat( szProtect, "Ex" );
		if( mbi.Protect & PAGE_EXECUTE_READ )
			strcat( szProtect, "Er" );
//		if( mbi.Type & PAGE_EXECUTE_WRITE ) NT only
//			strcat( szType, "Ew" );
		if( mbi.Protect & PAGE_EXECUTE_READWRITE )
			strcat( szProtect, "Erw" );
		if( mbi.Protect & PAGE_EXECUTE_WRITECOPY )
			strcat( szProtect, "Ewc" );
		if( mbi.Protect & PAGE_GUARD )
			strcat( szProtect, "G" );
		if( mbi.Protect & PAGE_NOACCESS )
			strcat( szProtect, "NA" );
		if( mbi.Protect & PAGE_NOCACHE )
			strcat( szProtect, "NC" );
		if( mbi.Type & MEM_IMAGE )
			strcat( szType, "I" );
		if( mbi.Type & MEM_MAPPED )
			strcat( szType, "M" );
		if( mbi.Type & MEM_PRIVATE )
			strcat( szType, "P" );
		wsprintf( szMessage,"Addr [0x%x] Alloc base [0x%x] Base [0x%x]"
			"RegSzK [%u] State [%s] Prot [%s] Type [%s]",
			lpAdd, mbi.AllocationBase, mbi.BaseAddress, mbi.RegionSize/1024,
			szState, szProtect, szType );
		srv_sendmsg( srvproc, SRV_MSG_INFO, XP_NOERROR, SRV_INFO, (DBTINYINT)0, NULL, 0, 0, szMessage, SRV_NULLTERM );
		srv_senddone( srvproc, SRV_DONE_MORE, 0, 0 );
		lpAdd += mbi.RegionSize ;
		szState[0] = '\0';
		szProtect[0] = '\0';
		szType[0] = '\0';
	}while( --iHops && lpAdd < (PBYTE)(si.lpMaximumApplicationAddress )
		&& lpAdd > lpBase );
	srv_senddone( srvproc, SRV_DONE_FINAL | SRV_DONE_COUNT, 0, iHops_c - iHops );
	return XP_NOERROR;
}
__declspec(dllexport) ULONG _GetXpVersion()
{
	return ODS_VERSION;
}
```

