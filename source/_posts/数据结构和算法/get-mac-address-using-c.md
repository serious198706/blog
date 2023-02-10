---
title: C++获取本机MAC地址
link: c++-get-mac
category:
  - Development
tag:
  - Linux
  - c/c++
date: 2012-12-21 00:00:00
---


最近出现一个需求，要获取本机的MAC地址，就找了一些资料。

<!-- more -->

### Windows

在Windows下，有两种方法可以获取MAC地址：

#### 1.通过SendARP函数来获取。

SendARP的函数原型：
``` c
DWORD SendARP(
  _In_     IPAddr DestIP,
  _In_     IPAddr SrcIP,
  _Out_    PULONG pMacAddr,
  _Inout_  PULONG PhyAddrLen
);
``` 

根据函数原型可以看出，可以通过IP地址来获取，也可以无视IP地址，这样就获得第一个网卡的MAC地址。

第三个参数pMacAddr与第四个参数PhyAddrLen类型相同，是指向ULONG类型数组的指针，可以通过强转为unsigned char *，用来显示等。
示例如下：

``` c
int GetOtherMacAddr(char *szIP,char *szBuf,int *pnBufLen)
{
	HRESULT hr;
	IPAddr  ipAddr;
	ULONG   pulMac[2];
	ULONG   ulLen;
	char strMacAddr[100]={0};
	ipAddr = inet_addr (szIP);
	memset (pulMac, 0xff, sizeof (pulMac));
	ulLen = 6;
	hr = SendARP (ipAddr, 0, pulMac, &ulLen);
	if(hr!=NO_ERROR)
		return 1;

	unsigned char * mac_addr=(unsigned char*)pulMac;
	sprintf(strMacAddr,"%02X:%02X:%02X:%02X:%02X:%02X",mac_addr[0],mac_addr[1],
		mac_addr[2],mac_addr[3],mac_addr[4],mac_addr[5]);
	if ( *pnBufLen <= (int)strlen(strMacAddr) )
		return 2;
	strcpy(szBuf,strMacAddr);
	*pnBufLen = strlen(szBuf);

	return 0;
}

int GetLocalMacAddr(char *szMac,int *pnMacLen,char *szIP /*=NULL */)
{
	//如果指定了IP，则直接按IP获取MAC
	//否则，需要先获取本机名称，再获取IP，再获取MAC
	if ( szIP != NULL )
		return GetOtherMacAddr(szIP,szMac,pnMacLen);

	char szHostName[256] = {0};
	int nRet = gethostname(szHostName,256);
	if ( nRet == SOCKET_ERROR )
		return 1;

	//获取本机名称
	struct hostent* hHost = gethostbyname(szHostName);
	if ( hHost == NULL ||  hHost->h_addr_list[0] == NULL )
		return 2;

	//获取IP地址
	memset(szHostName,0,256);
	strcpy(szHostName,inet_ntoa(*(struct in_addr *)hHost->h_addr_list[0]));

	//获取MAC
	return  GetOtherMacAddr(szHostName,szMac,pnMacLen);
}
``` 

#### 2.通过PIP_ADAPTER_INFO结构体与GetAdaptersInfo函数来获取

PIP_ADAPTER_INFO结构体的结构如下：
``` c
typedef struct _IP_ADAPTER_INFO {
	struct _IP_ADAPTER_INFO* Next;
	DWORD ComboIndex;
	char AdapterName[MAX_ADAPTER_NAME_LENGTH + 4];
	char Description[MAX_ADAPTER_DESCRIPTION_LENGTH + 4];
	UINT AddressLength;
	BYTE Address[MAX_ADAPTER_ADDRESS_LENGTH];
	DWORD Index;
	UINT Type;
	UINT DhcpEnabled;
	PIP_ADDR_STRING CurrentIpAddress;
	IP_ADDR_STRING IpAddressList;
	IP_ADDR_STRING GatewayList;
	IP_ADDR_STRING DhcpServer;
	BOOL HaveWins;
	IP_ADDR_STRING PrimaryWinsServer;
	IP_ADDR_STRING SecondaryWinsServer;
	time_t LeaseObtained;
	time_t LeaseExpires;
} IP_ADAPTER_INFO, *PIP_ADAPTER_INFO;
``` 

可以看到，有网卡名称、描述信息、MAC地址、IP地址列表、DHCP服务器等信息。这次我们用到的就是MAC地址BYTE[MAX_ADAPTER_ADDRESS_LENGTH。而struct _IP_ADAPTER_INFO* Next则告诉我们所有的网卡信息将会构成一个链表，通过Next来得到下一个网卡的信息。

``` c
void GetMAC(BSTR* pVal)
{
	CString sMAC;
	PIP_ADAPTER_INFO pAdapterInfo;
	PIP_ADAPTER_INFO pAdapter = NULL;
	DWORD dwRetVal = 0;
	ULONG ulOutBufLen;
	pAdapterInfo=(PIP_ADAPTER_INFO)malloc(sizeof(IP_ADAPTER_INFO));
	ulOutBufLen = sizeof(IP_ADAPTER_INFO);

	if (GetAdaptersInfo(pAdapterInfo, &ulOutBufLen) == ERROR_BUFFER_OVERFLOW)
	{
		free(pAdapterInfo);
		pAdapterInfo = (IP_ADAPTER_INFO *) malloc (ulOutBufLen);
	}

	if ((dwRetVal = GetAdaptersInfo(pAdapterInfo, &ulOutBufLen)) == NO_ERROR)
	{
		pAdapter = pAdapterInfo;
		if (pAdapter){
			sMAC.Format(_T("本机MAC地址为：%02x-%02x-%02x-%02x-%02x-%02x"),
				pAdapter->Address[0],
				pAdapter->Address[1],
				pAdapter->Address[2],
				pAdapter->Address[3],
				pAdapter->Address[4],
				pAdapter->Address[5]);
			//pAdapter = pAdapter->Next;  //如果有多网卡，这儿就可以使用此条语句转到下一个网卡
		}
	}

	*pVal = ::SysAllocString(sMAC.MakeUpper());
}
``` 

### Linux

在Linux下也类似。有一个名为ifreq的结构体，包含了网卡的所有信息。
ifreq的结构如下：

``` c
struct ifreq
{
	char ifr_name[IFNAMSIZ];
	union
	{
		struct sockaddr ifru_addr;
		struct sockaddr ifru_dstaddr;
		struct sockaddr ifru_broadaddr;
		struct sockaddr ifru_netmask;
		struct sockaddr ifru_hwaddr;
		short int ifru_flags;
		int ifru_ivalue;
		int ifru_mtu;
		struct ifmap ifru_map;
		char ifru_slave[IFNAMSIZ]; /* Just fits the size */
		char ifru_newname[IFNAMSIZ];
		__caddr_t ifru_data;
	} ifr_ifru;
};

# define ifr_name ifr_ifrn.ifrn_name /* interface name */
# define ifr_hwaddr ifr_ifru.ifru_hwaddr /* MAC address */
# define ifr_addr ifr_ifru.ifru_addr /* address */
# define ifr_dstaddr ifr_ifru.ifru_dstaddr /* other end of p-p lnk */
# define ifr_broadaddr ifr_ifru.ifru_broadaddr /* broadcast address */
# define ifr_netmask ifr_ifru.ifru_netmask /* interface net mask */
# define ifr_flags ifr_ifru.ifru_flags /* flags */
# define ifr_metric ifr_ifru.ifru_ivalue /* metric */
# define ifr_mtu ifr_ifru.ifru_mtu /* mtu */
# define ifr_map ifr_ifru.ifru_map /* device map */
# define ifr_slave ifr_ifru.ifru_slave /* slave device */
# define ifr_data ifr_ifru.ifru_data /* for use by interface */
# define ifr_ifindex ifr_ifru.ifru_ivalue /* interface index */
# define ifr_bandwidth ifr_ifru.ifru_ivalue /* link bandwidth */
# define ifr_qlen ifr_ifru.ifru_ivalue /* queue length */
# define ifr_newname ifr_ifru.ifru_newname /* New name */
# define _IOT_ifreq _IOT(_IOTS(char),IFNAMSIZ,_IOTS(char),16,0,0)
# define _IOT_ifreq_short _IOT(_IOTS(char),IFNAMSIZ,_IOTS(short),1,0,0)
# define _IOT_ifreq_int _IOT(_IOTS(char),IFNAMSIZ,_IOTS(int),1,0,0)
``` 

稍微复杂点，但还是能找到我们所需要的信息的 - ifru_hwaddr。

通过下面的代码，就能完成工作了：

``` c
int GetLocalMacAddr(char *szMac,int *pnMacLen)
{
	int   sock;
	struct   ifreq   ifr;
	unsigned   char   mac[6];

	sock=socket(AF_INET,SOCK_DGRAM,0);    // 生成一个TCP的socket
	if (sock==-1)
	{
		perror("socket");
		return 1;
	}

	strncpy(ifr.ifr_name,"eth0",sizeof(ifr.ifr_name));
	ifr.ifr_name[IFNAMSIZ-1]   =   0;

	memset(mac,0,sizeof(mac));
	if (ioctl(sock,SIOCGIFHWADDR,&ifr)< 0)
	{
		perror("ioctl");
		return 2;
	}

	memcpy(mac,&ifr.ifr_hwaddr.sa_data,sizeof(mac));
	char curmacstr[64];
	memset(curmacstr,0,sizeof(curmacstr));
	sprintf(curmacstr,"%.2X:%.2X:%.2X:%.2X:%.2X:%.2X",mac[0],mac[1],mac[2],mac[3],mac[4],mac[5]);
	strcpy(szMac,curmacstr);
	return 0;
}
``` 
