
```c
#include <Windows.h>
#include <winsock.h>
#include <stdio.h>
#include <stdlib.h>

#pragma comment(lib , "Ws2_32.lib")

typedef struct sockaddr_in sockaddr_in;
typedef struct sockaddr sockaddr;

typedef struct {
	SOCKET clientSock;
	sockaddr_in clientInfo;
}client;

BOOL initializeWSA() {
	WSADATA wsData;
	if (WSAStartup(MAKEWORD(2, 2), &wsData) == SOCKET_ERROR) {
		printf("Failed to initialize WSA");
		return 0;
	};
	return 1;
}

SOCKET initializeServer() {
	if (!initializeWSA()) {
		return NULL;
	}

	SOCKET serverSock;
	sockaddr_in serverInfo;

	serverSock = socket(AF_INET, SOCK_STREAM, 0);
	if (serverSock == INVALID_SOCKET) {
		printf("Couldnt create socket");
		return NULL;
	};

	serverInfo.sin_addr.S_un.S_addr = INADDR_ANY;
	serverInfo.sin_family = AF_INET;
	serverInfo.sin_port = htons(9999);

	if (bind(serverSock, (sockaddr*)&serverInfo, sizeof(serverInfo)) == SOCKET_ERROR) {
		printf("Couldnt bind socket");
		closesocket(serverSock);
		return NULL;
	}

	printf("[!] Server created with ip: %s , port: %hu\n",
		inet_ntoa(serverInfo.sin_addr), ntohs(serverInfo.sin_port));

	return serverSock;
}

DWORD handleClient(client Client);

void runServer(SOCKET serverSock) {
	SOCKET clientSock;
	sockaddr_in clientInfo;
	int clientStSz = sizeof(clientInfo);
	int clientCounter = 0;

	__try {
		if (listen(serverSock, 20) == SOCKET_ERROR) {
			printf("server couldnt listen");
			return;
		}

		printf("[+] Server running with 20 client queue size\n");

		while (1) {
			printf("[!] Waiting for clients...\n");

			clientSock = accept(serverSock, (sockaddr*)&clientInfo, &clientStSz);
			if (clientSock == INVALID_SOCKET) {
				printf("couldnt get client socket");
				return;
			}

			printf("[+] Client connected ip: %s , port: %hu\n",
				inet_ntoa(clientInfo.sin_addr), ntohs(clientInfo.sin_port));

			client Client = {
				.clientInfo = clientInfo,
				.clientSock = clientSock
			};

			CreateThread(
				NULL,
				0,
				handleClient,
				&Client,
				0,
				NULL
			);

			printf("[!] Thread created to handle Client , id: %d\n", clientCounter++);
		}
	}
	__finally {
		closesocket(serverSock);
	}
}

void serverSendClient();
void serverRecvClient();

DWORD handleClient(client Client){
	send(Client.clientSock, "Hello world", 12, 0);

	closesocket(Client.clientSock);
}

int main() {
	SOCKET server = initializeServer();
	runServer(server);

	return 0;
}
```

---
