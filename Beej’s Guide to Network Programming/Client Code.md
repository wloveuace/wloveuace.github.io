
```c
#include <Windows.h>
#include <stdio.h>
#include <stdlib.h>
#include <winsock.h>

#pragma comment(lib, "Ws2_32.lib")

typedef struct sockaddr_in sockaddr_in;
typedef struct sockaddr sockaddr;

BOOL initializeWSA() {
	WSADATA wsData;
	if (WSAStartup(MAKEWORD(2, 2), &wsData) == SOCKET_ERROR) {
		printf("Failed to initialize WSA");
		return 0;
	};
	return 1;
}

SOCKET initializeClient() {
	if (!initializeWSA()) {
		return NULL;
	}

	SOCKET clientSock;

	clientSock = socket(AF_INET, SOCK_STREAM, 0);
	if (clientSock == INVALID_SOCKET) {
		printf("couldnt create client socket");
		return NULL;
	}

	printf("[+] Client ready to connect\n");

	return clientSock;
}

BOOL connectClientToServer(SOCKET clientSock) {
	if (clientSock == NULL) return 0 ;

	sockaddr_in serverInfo = {
		.sin_addr.S_un.S_addr = inet_addr("127.0.0.1"),
		.sin_family = AF_INET,
		.sin_port = htons(9999)
	};

		printf("[!] Connecting to server\n");

		if (connect(clientSock, (sockaddr*)&serverInfo, sizeof(serverInfo)) == INVALID_SOCKET) {
			printf("Failed to connect to server");
			closesocket(clientSock);
			return 0;
		};

		printf("[+] Connected to server with ip: %s , port %hu\n",
			inet_ntoa(serverInfo.sin_addr), ntohs(serverInfo.sin_port));

		return 1;
}

void clientRecvFromServer(SOCKET clientSock) {
	while (1) {
		char buffer[100];
		int r = recv(clientSock, buffer, 100, 0);
		if (r == 0) continue;

		buffer[r] = '\0';
		printf("buffer: %s\n", buffer);

		memset(buffer, 0, 100);
	}
}

int main() {
	SOCKET clientSock = initializeClient();
	BOOL Connection = connectClientToServer(clientSock);
	if (!Connection) return;

	clientRecvFromServer(clientSock);
	closesocket(clientSock);
}
```

---
