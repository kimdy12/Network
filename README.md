파일을 주고받는 서버

    #include <stdio.h>
    #include <stdlib.h>
    #include <string>
    #include <winsock2.h>
    #include <ws2tcpip.h>
    #include <fstream>
    #include "..\..\Common.h"   // err_quit(), err_display() 정의된 헤더
    
    #pragma comment(lib, "ws2_32.lib")  // 윈속 라이브러리 링크
    
    #define SERVERPORT 9000
    #define BUFSIZE    512
    using namespace std;
  
      int main(int argc, char* argv[])
      {
          int retval;
      
          WSADATA wsa;
          if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0)
              return 1;
      
          SOCKET listen_sock = socket(AF_INET, SOCK_STREAM, 0);
          if (listen_sock == INVALID_SOCKET) err_quit("socket()");
      
          struct sockaddr_in serveraddr;
          memset(&serveraddr, 0, sizeof(serveraddr));
          serveraddr.sin_family = AF_INET;
          serveraddr.sin_addr.s_addr = htonl(INADDR_ANY);
          serveraddr.sin_port = htons(SERVERPORT);
          retval = bind(listen_sock, (struct sockaddr*)&serveraddr, sizeof(serveraddr));
          if (retval == SOCKET_ERROR) err_quit("bind()");
      
          retval = listen(listen_sock, SOMAXCONN);
          if (retval == SOCKET_ERROR) err_quit("listen()");
      
          SOCKET client_sock;
          struct sockaddr_in clientaddr;
          int addrlen;
          char buf[BUFSIZE + 1];
      
          while (1) {
              printf("연결을 기다리는 중\n");
      
              addrlen = sizeof(clientaddr);
              client_sock = accept(listen_sock, (struct sockaddr*)&clientaddr, &addrlen);
              if (client_sock == INVALID_SOCKET) {
                  err_display("accept()");
                  break;
              }
      
              char addr[INET_ADDRSTRLEN];
              inet_ntop(AF_INET, &clientaddr.sin_addr, addr, sizeof(addr));
              printf("\n[TCP 서버] 클라이언트 접속: IP=%s, 포트=%d\n",
                  addr, ntohs(clientaddr.sin_port));
      
              while (1) {
                  retval = recv(client_sock, buf, BUFSIZE, 0);
                  if (retval == SOCKET_ERROR) { err_display("recv()"); break; }
                  else if (retval == 0) break;
      
                  buf[retval] = '\0';
                  printf("[TCP/%s:%d] 클라이언트 메시지: %s\n",
                      addr, ntohs(clientaddr.sin_port), buf);
      
                  if (strcmp(buf, "list") == 0) {
                      const char* listMsg =
                          "a.txt\n"
                          "b.txt\n"
                          "c.png\n"
                          "d.png\n";
                      send(client_sock, listMsg, (int)strlen(listMsg) + 1, 0); // null 포함
                      continue;
                  }
      
                  if (strncmp(buf, "get ", 4) == 0) {
                      string filename(buf + 4);
      
                      ifstream file(filename, ios::binary);
                      if (!file) {
                          const char* noneMsg = "none\0";
                          send(client_sock, noneMsg, (int)strlen(noneMsg) + 1, 0);
                          closesocket(client_sock); // 클라이언트 종료
                          printf("[TCP 서버] 파일 없음, 클라이언트 종료: IP=%s, 포트=%d\n",
                              addr, ntohs(clientaddr.sin_port));
                          break;
                      }
      
                      char filebuf[BUFSIZE];
                      while (!file.eof()) {
                          file.read(filebuf, sizeof(filebuf));
                          int bytesRead = (int)file.gcount();
                          send(client_sock, filebuf, bytesRead, 0);
                      }
                      file.close();
      
                      const char* doneMsg = "\n[SERVER] File transfer complete\0";
                      send(client_sock, doneMsg, (int)strlen(doneMsg) + 1, 0);
      
                      closesocket(client_sock);
                      printf("[TCP 서버] 파일 전송 완료, 클라이언트 종료: IP=%s, 포트=%d\n",
                          addr, ntohs(clientaddr.sin_port));
                      break;
                  }
      
                  retval = send(client_sock, buf, retval, 0);
                  if (retval == SOCKET_ERROR) { err_display("send()"); break; }
              }
      
              if (client_sock != INVALID_SOCKET) {
                  closesocket(client_sock);
              }
          }
      
          closesocket(listen_sock);
          WSACleanup();
          return 0;
      }

파일을 주고받는 클라이언트

      #include <stdio.h>
      #include <stdlib.h>
      #include <string>
      #include <winsock2.h>
      #include <ws2tcpip.h>
      #include <fstream>
      #include <iostream>
      
      #pragma comment(lib, "ws2_32.lib")
      
      #define SERVERIP   "127.0.0.1" // 서버 IP
      #define SERVERPORT 9000
      #define BUFSIZE    512
      
      using namespace std;
      
      int main() {
          WSADATA wsa;
          if (WSAStartup(MAKEWORD(2, 2), &wsa) != 0) return 1;
      
          SOCKET sock = socket(AF_INET, SOCK_STREAM, 0);
          if (sock == INVALID_SOCKET) {
              printf("socket() 실패\n");
              WSACleanup();
              return 1;
          }
      
          struct sockaddr_in serveraddr;
          memset(&serveraddr, 0, sizeof(serveraddr));
          serveraddr.sin_family = AF_INET;
          inet_pton(AF_INET, SERVERIP, &serveraddr.sin_addr);
          serveraddr.sin_port = htons(SERVERPORT);
      
          if (connect(sock, (struct sockaddr*)&serveraddr, sizeof(serveraddr)) == SOCKET_ERROR) {
              printf("connect() 실패\n");
              closesocket(sock);
              WSACleanup();
              return 1;
          }
      
          char buf[BUFSIZE + 1];
          string cmd;
      
          while (true) {
              printf("\n보낼 메시지: ");
              getline(cin, cmd);
      
              if (cmd == "quit") break;
      
              send(sock, cmd.c_str(), (int)cmd.size() + 1, 0);
      
              if (cmd == "list") {
                  int n = recv(sock, buf, BUFSIZE, 0);
                  if (n > 0) {
                      buf[n] = '\0';
                      printf("서버 파일 리스트:\n%s\n", buf);
                  }
                  continue;
              }
      
              if (cmd.substr(0, 4) == "get ") {
                  string filename = cmd.substr(4);
                  ofstream outFile(filename, ios::binary);
                  int n;
      
                  while ((n = recv(sock, buf, BUFSIZE, 0)) > 0) {
                      buf[n] = '\0';
                      string msg(buf, n);
      
                      if (msg == "none") {
                          printf("[클라이언트] 서버에 파일이 없습니다.\n");
                          outFile.close();
                          remove(filename.c_str());
                          break;
                      }
                      if (msg.find("[SERVER] File transfer complete") != string::npos) {
                          printf("[클라이언트] 파일 전송 완료\n");
                          break;
                      }
      
                      outFile.write(buf, n);
                  }
                  outFile.close();
                  continue;
              }
      
              printf("[클라이언트] 알 수 없는 명령입니다.\n");
          }
      
          closesocket(sock);
          WSACleanup();
          return 0;
      }
