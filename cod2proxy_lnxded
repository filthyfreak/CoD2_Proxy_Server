// Copyright (c) 2023, king-clan.com
// All rights reserved.

// This source code is licensed under the BSD-style license found in the
// LICENSE file in the root directory of this source tree. 

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <pthread.h>
#include <signal.h>
#include <errno.h>
#include <netdb.h>
#include <sys/time.h>

#define TIMEOUT_SECONDS 240
#define MAX_BUFFER_SIZE 65536

char HEARTBEAT_MESSAGE[] = "\xFF\xFF\xFF\xFFheartbeat COD-2";
char AUTHORIZE_MESSAGE[] = "\xFF\xFF\xFF\xFFgetIpAuthorize";
char SHORTVERSION[4];
char FORWARD_IP[] = "127.0.0.1";
char FS_GAME[32];
struct sockaddr_in listen_addr;
struct sockaddr_in forward_addr;
int sock_dict_size = 0;
int BLOCKIPS;
int FORWARD_PORT;
int LISTEN_PORT;
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

// These IP addresses are website server trackers. If we don't want our proxy server to respond to them, add them to this array and make sure BLOCKIPS(BOOL) is true in startup command.
char *blocked_ips[] = {"208.167.241.187", "108.61.78.150", "108.61.78.149", "149.28.43.230", "45.77.96.90", "159.69.0.99", "178.254.22.101", "63.239.170.80", "63.239.170.18", "63.239.170.8", "159.146.13.56", "217.131.98.94", "78.46.106.94"};

typedef struct
{
	int s_client;
	pthread_t thread;
} ThreadInfo;

typedef struct
{
	int s_server;
} MasterThreadArgs;

typedef struct
{
	struct sockaddr_in addr;
	int src_port;
	int s_server;
	int activeClient;
	int *s_client;
} ListenThreadArgs;

typedef struct
{
	int type;
	char ip[4];
	unsigned short port;
	char ipx[10];
} netadr_t;

typedef struct leakyBucket_s leakyBucket_t;
struct leakyBucket_s
{
	int type;
	unsigned char adr[4];
	uint64_t lastTime;
	signed char	burst;
	long hash;
	leakyBucket_t *prev, *next;
};

unsigned long hashString(const char *str)
{
	unsigned long hash = 5381;
	int c;

	while ((c = *str++))
		hash = ((hash << 5) + hash) + c; // djb2 algorithm

	return hash;
}

uint64_t createClientIdentifier(struct sockaddr_in *addr)
{
	char client_info[INET_ADDRSTRLEN + 6];
	uint32_t client_address = ntohl(addr->sin_addr.s_addr);
	uint16_t client_port = ntohs(addr->sin_port);

	snprintf(client_info, sizeof(client_info), "%u:%u", client_address, client_port);

	return (uint64_t)hashString(client_info);
}

uint32_t hash(uint64_t identifier, uint32_t array_size)
{
	return (uint32_t)(identifier % array_size);
}

void SockadrToNetadr (struct sockaddr_in *s, netadr_t *a)
{
	*(int *)&a->ip = *(int *)&s->sin_addr;
	a->port = s->sin_port;
	a->type = 4;
}

time_t sys_timeBase64 = 0;
uint64_t Sys_Milliseconds64(void)
{
	struct timeval tp;

	gettimeofday(&tp, NULL);

	if (!sys_timeBase64)
	{
		sys_timeBase64 = tp.tv_sec;
		return tp.tv_usec / 1000;
	}

	return (tp.tv_sec - sys_timeBase64) * 1000 + tp.tv_usec / 1000;
}

// ioquake3 rate limit connectionless requests
// https://github.com/ioquake/ioq3/blob/master/code/server/sv_main.c

// This is deliberately quite large to make it more of an effort to DoS
#define MAX_BUCKETS	16384
#define MAX_HASHES 1024

static leakyBucket_t buckets[MAX_BUCKETS];
static leakyBucket_t* bucketHashes[MAX_HASHES];
leakyBucket_t outboundLeakyBucket;

static long SVC_HashForAddress(netadr_t address)
{
	unsigned char *ip = address.ip;
	int	i;
	long hash = 0;

	for( i = 0; i < 4; i++)
		hash += (long)( ip[ i ] ) * ( i + 119 );

	hash = (hash ^ (hash >> 10) ^ (hash >> 20));
	hash &= (MAX_HASHES - 1);

	return hash;
}

static leakyBucket_t *SVC_BucketForAddress(netadr_t address, int burst, int period)
{
	leakyBucket_t *bucket = NULL;
	int	i;
	long hash = SVC_HashForAddress(address);
	uint64_t now = Sys_Milliseconds64();

	for(bucket = bucketHashes[hash]; bucket; bucket = bucket->next)
	{
		if (memcmp(bucket->adr, address.ip, 4) == 0)
			return bucket;
	}

	for(i = 0; i < MAX_BUCKETS; i++)
	{
		int interval;

		bucket = &buckets[ i ];
		interval = now - bucket->lastTime;

		// Reclaim expired buckets
		if(bucket->lastTime > 0 && (interval > (burst * period) || interval < 0))
		{
			if(bucket->prev != NULL)
				bucket->prev->next = bucket->next;
			else
				bucketHashes[bucket->hash] = bucket->next;

			if(bucket->next != NULL)
				bucket->next->prev = bucket->prev;

			memset(bucket, 0, sizeof(leakyBucket_t));
		}

		if(bucket->type == 0)
		{
			bucket->type = address.type;
			memcpy(bucket->adr, address.ip, 4);

			bucket->lastTime = now;
			bucket->burst = 0;
			bucket->hash = hash;

			// Add to the head of the relevant hash chain
			bucket->next = bucketHashes[hash];
			if(bucketHashes[hash] != NULL)
				bucketHashes[hash]->prev = bucket;

			bucket->prev = NULL;
			bucketHashes[hash] = bucket;

			return bucket;
		}
	}

	// Couldn't allocate a bucket for this address
	return NULL;
}

int SVC_RateLimit(leakyBucket_t *bucket, int burst, int period)
{
	if (bucket != NULL)
	{
		uint64_t now = Sys_Milliseconds64();
		int interval = now - bucket->lastTime;
		int expired = interval / period;
		int expiredRemainder = interval % period;

		if (expired > bucket->burst || interval < 0)
		{
			bucket->burst = 0;
			bucket->lastTime = now;
		}
		else
		{
			bucket->burst -= expired;
			bucket->lastTime = now - expiredRemainder;
		}

		if(bucket->burst < burst)
		{
			bucket->burst++;

			return 0;
		}
	}

	return 1;
}

int SVC_RateLimitAddress(netadr_t from, int burst, int period)
{
	leakyBucket_t *bucket = SVC_BucketForAddress(from, burst, period);

	return SVC_RateLimit(bucket, burst, period);
}

void *master_thread(void *arg)
{
	MasterThreadArgs *args = (MasterThreadArgs *)arg;

	const char *hostname = "cod2master.activision.com";
	struct addrinfo hints;
	memset(&hints, 0, sizeof(hints));
	hints.ai_family = AF_INET;
	hints.ai_socktype = SOCK_DGRAM;

	struct addrinfo *result;
	int status = getaddrinfo(hostname, NULL, &hints, &result);
	if(status != 0)
	{
		fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(status));
		exit(1);
	}

	struct sockaddr_in *master_addr = (struct sockaddr_in *)(result->ai_addr);
	uint16_t heartbeat_port = 20710;
	uint16_t authorize_port = 20700;

	while(1)
	{
		printf("Sending getIpAuthorize to %s:%d (%s)\n", inet_ntoa(master_addr->sin_addr), authorize_port, "cod2master.activision.com");
		char AUTHORIZE_SEND[MAX_BUFFER_SIZE];
		sprintf(AUTHORIZE_SEND, "%s %ld %s %s 0", AUTHORIZE_MESSAGE, time(NULL), inet_ntoa(master_addr->sin_addr), FS_GAME);
		master_addr->sin_port = htons(authorize_port);
		sendto(args->s_server, AUTHORIZE_SEND, strlen(AUTHORIZE_SEND), 0, result->ai_addr, result->ai_addrlen);

		printf("Sending heartbeat to %s:%d (%s)\n", inet_ntoa(master_addr->sin_addr), heartbeat_port, "cod2master.activision.com");
		master_addr->sin_port = htons(heartbeat_port);
		sendto(args->s_server, HEARTBEAT_MESSAGE, strlen(HEARTBEAT_MESSAGE), 0, result->ai_addr, result->ai_addrlen);

		sleep(60);
	}

	freeaddrinfo(result); // Cleanup
	return NULL;
}

void *listen_thread(void *arg)
{
	ListenThreadArgs *args = (ListenThreadArgs *)arg;
	char client_ip[INET_ADDRSTRLEN];

	while(1)
	{
		char buffer[MAX_BUFFER_SIZE];
		struct sockaddr_in r_addr;
		socklen_t r_len = sizeof(r_addr);

		ssize_t bytes_received = recvfrom(*(args->s_client), buffer, sizeof(buffer) - 1, 0, (struct sockaddr *)&r_addr, &r_len);

		inet_ntop(AF_INET, &args->addr.sin_addr, client_ip, sizeof(client_ip));

		if(bytes_received >= 0)
		{
			if(bytes_received < sizeof(buffer) - 1)
				buffer[bytes_received] = '\0';
			else
			{
				perror("recvfrom max size exceeded");
				continue;
			}
		}
		else
		{
			if(errno == EAGAIN || errno == EWOULDBLOCK)
				break;
			else
			{
				perror("recvfrom listen_thread no data");
				continue;
			}
		}

		if(memcmp(buffer, "\xFF\xFF\xFF\xFFstatusResponse", 18) == 0 || memcmp(buffer, "\xFF\xFF\xFF\xFFinfoResponse", 16) == 0)
		{
			int is_blocked = 0;
			for(int i = 0; i < sizeof(blocked_ips) / sizeof(blocked_ips[0]); i++)
			{
				if(strcmp(client_ip, blocked_ips[i]) == 0)
				{
					is_blocked = 1;
					break;
				}
			}

			if(!is_blocked || !BLOCKIPS)
			{
				if(memcmp(buffer, "\xFF\xFF\xFF\xFFstatusResponse", 18) == 0)
				{
					if(strcmp(SHORTVERSION, "1.2") == 0)
					{
						char *protocol = strstr(buffer, "\\protocol\\115");
						if(protocol != NULL)
							memcpy(protocol, "\\protocol\\117", 13);

						char *shortversion = strstr(buffer, "\\shortversion\\1.0");
						if(shortversion != NULL)
							memcpy(shortversion, "\\shortversion\\1.2", 17);
					}
					else
					{
						char *protocol = strstr(buffer, "\\protocol\\115");
						if(protocol != NULL)
							memcpy(protocol, "\\protocol\\118", 13);

						char *shortversion = strstr(buffer, "\\shortversion\\1.0");
						if(shortversion != NULL)
							memcpy(shortversion, "\\shortversion\\1.3", 17);
					}
				}
				else
				{
					if(strcmp(SHORTVERSION, "1.2") == 0)
					{
						char *protocol = strstr(buffer, "\\protocol\\115");
						if(protocol != NULL)
							memcpy(protocol, "\\protocol\\117", 13);
					}
					else
					{
						char *protocol = strstr(buffer, "\\protocol\\115");
						if(protocol != NULL)
							memcpy(protocol, "\\protocol\\118", 13);
					}
				}
			}
			else
			{
				strcpy(buffer, "\xFF\xFF\xFF\xFF""disconnect");
				bytes_received = strlen(buffer);
				buffer[bytes_received] = '\0';
			}
		}

		// Forward packets to the client
		ssize_t sent_bytes = sendto(args->s_server, buffer, bytes_received, 0, (struct sockaddr *)&args->addr, sizeof(args->addr));

		if(sent_bytes == -1)
			perror("sendto");
	}

	if(strlen(client_ip) && args->activeClient)
		printf("Client connection released for %s:%d\n", client_ip, ntohs(args->addr.sin_port));

	pthread_mutex_lock(&lock);
	close(*(args->s_client));
	*(args->s_client) = -1;
	sock_dict_size--;
	pthread_mutex_unlock(&lock);

	free(args);
	return NULL;
}

void forceful_exit(int signum)
{
	exit(0);
}

void toLowerCase(char *str)
{
	for (int i = 0; str[i]; i++)
	{
		if (str[i] >= 'A' && str[i] <= 'Z')
			str[i] = str[i] + ('a' - 'A');
	}
}

void *input_thread(void *arg)
{
	char user_input[256];
	while(1)
	{
		fgets(user_input, sizeof(user_input), stdin);
		toLowerCase(user_input);

		if(strcmp(user_input, "quit\n") == 0)
			exit(0);
	}
	return NULL;
}

int main(int argc, char *argv[])
{
	if(argc != 6)
	{
		printf("Usage: %s <FORWARD_TO> <LISTEN_ON> <SHORTVERSION> <FS_GAME> <BLOCKIPS(BOOL)>\n", argv[0]);
		exit(1);
	}

	FORWARD_PORT = atoi(argv[1]);
	LISTEN_PORT = atoi(argv[2]);
	strncpy(SHORTVERSION, argv[3], sizeof(SHORTVERSION));
	strncpy(FS_GAME, argv[4], sizeof(FS_GAME));
	BLOCKIPS = atoi(argv[5]);

	memset(&listen_addr, 0, sizeof(listen_addr));
	listen_addr.sin_family = AF_INET;
	listen_addr.sin_port = htons(LISTEN_PORT);
	listen_addr.sin_addr.s_addr = INADDR_ANY;
	printf("CoD2 v%s proxy server listening on *:%d\n", SHORTVERSION, LISTEN_PORT);

	memset(&forward_addr, 0, sizeof(forward_addr));
	forward_addr.sin_family = AF_INET;
	forward_addr.sin_port = htons(FORWARD_PORT);
	inet_pton(AF_INET, FORWARD_IP, &forward_addr.sin_addr);

	int sock = socket(AF_INET, SOCK_DGRAM, 0);
	if(sock == -1)
	{
		perror("socket");
		exit(1);
	}
	if(bind(sock, (struct sockaddr *)&listen_addr, sizeof(listen_addr)) == -1)
	{
		perror("bind");
		close(sock);
		exit(1);
	}

	signal(SIGINT, forceful_exit);

	MasterThreadArgs *args_ = (MasterThreadArgs *)malloc(sizeof(MasterThreadArgs));
	args_->s_server = sock;

	pthread_t m_thread, input_t;
	pthread_create(&m_thread, NULL, master_thread, args_);
	pthread_create(&input_t, NULL, input_thread, NULL);
	pthread_detach(m_thread);
	pthread_detach(input_t);

	ThreadInfo sock_dict[65536];
	memset(sock_dict, -1, sizeof(sock_dict));

	while(1)
	{
		char buffer[MAX_BUFFER_SIZE];
		char lowerCaseBuffer[MAX_BUFFER_SIZE];
		struct sockaddr_in addr;
		netadr_t adr;
		socklen_t addr_len = sizeof(addr);

		ssize_t bytes_received = recvfrom(sock, buffer, sizeof(buffer) - 1, 0, (struct sockaddr *)&addr, &addr_len);
		if(bytes_received >= 0) {
			if (bytes_received < sizeof(buffer) - 1)
				buffer[bytes_received] = '\0';
			else
			{
				perror("recvfrom max size exceeded");
				continue;
			}
		}
		else
		{
			perror("recvfrom no data");
			continue;
		}

		SockadrToNetadr(&addr, &adr);

		int activeClient = 1;
		char client_ip[INET_ADDRSTRLEN];
		inet_ntop(AF_INET, &addr.sin_addr, client_ip, sizeof(client_ip));

		strcpy(lowerCaseBuffer, buffer);
		toLowerCase(lowerCaseBuffer);

		if(memcmp(lowerCaseBuffer, "\xff\xff\xff\xffgetstatus", 13) == 0)
		{
			// Prevent using getstatus as an amplifier
			if(SVC_RateLimitAddress(adr, 10, 1000))
				continue;

			activeClient = 0;
			int is_blocked = 0;
			for(int i = 0; i < sizeof(blocked_ips) / sizeof(blocked_ips[0]); i++)
			{
				if(strcmp(client_ip, blocked_ips[i]) == 0)
				{
					is_blocked = 1;
					break;
				}
			}

			if(is_blocked)
			{
				strcpy(buffer, "\xFF\xFF\xFF\xFFgetstatusproxy");
				bytes_received = bytes_received + 5;
			}
		}
		else if(memcmp(lowerCaseBuffer, "\xff\xff\xff\xff""connect", 11) == 0)
		{
			size_t current_len = strlen(buffer);
			char ip_insertion[] = "\\ip\\";

			if(current_len + strlen(ip_insertion) + strlen(client_ip) + 1 < MAX_BUFFER_SIZE)
			{
				size_t insert_position = current_len - 1;
				memmove(buffer + insert_position + strlen(ip_insertion), buffer + insert_position, current_len - insert_position);
				memcpy(buffer + insert_position, ip_insertion, strlen(ip_insertion));
				memcpy(buffer + insert_position + strlen(ip_insertion), client_ip, strlen(client_ip));
				memcpy(buffer + insert_position + strlen(ip_insertion) + strlen(client_ip), "\"", 1);
				bytes_received = bytes_received + strlen(ip_insertion) + strlen(client_ip) + 1;
			}
			else
			{
				perror("memmove");
				continue;
			}
		}
		else if(memcmp(lowerCaseBuffer, "\xff\xff\xff\xffgetchallenge", 16) == 0 || memcmp(lowerCaseBuffer, "\xff\xff\xff\xffgetinfo", 11) == 0 || memcmp(lowerCaseBuffer, "\xff\xff\xff\xffrcon", 8) == 0)
		{
			// Prevent using getchallenge/getinfo/rcon as an amplifier
			if(SVC_RateLimitAddress( adr, 10, 1000))
				continue;

			if(memcmp(lowerCaseBuffer, "\xff\xff\xff\xffgetchallenge 0 \"", 20) != 0)
				activeClient = 0;
		}

		pthread_mutex_lock(&lock);

		uint64_t client_identifier = createClientIdentifier(&addr);
		uint32_t index_value = hash(client_identifier, sizeof(sock_dict) / sizeof(sock_dict[0]));

		if(index_value >= 0 && index_value < sizeof(sock_dict) / sizeof(sock_dict[0]))
		{
			if(sock_dict_size == 0 || sock_dict[index_value].s_client == -1)
			{
				int s_client = socket(AF_INET, SOCK_DGRAM, 0);

				if(s_client == -1)
				{
					perror("socket");
					pthread_mutex_unlock(&lock);
					continue;
				}

				struct timeval tv;
				tv.tv_sec = TIMEOUT_SECONDS;
				tv.tv_usec = 0;
				setsockopt(s_client, SOL_SOCKET, SO_RCVTIMEO, (const char*)&tv, sizeof tv);

				ThreadInfo t_info;
				t_info.s_client = s_client;
				sock_dict[index_value] = t_info;
				sock_dict_size++;

				if(activeClient)
					printf("Client connecting from %s:%d\n", client_ip, ntohs(addr.sin_port));

				sendto(s_client, buffer, bytes_received, 0, (struct sockaddr *)&forward_addr, sizeof(forward_addr));

				ListenThreadArgs *args = (ListenThreadArgs *)malloc(sizeof(ListenThreadArgs));
				args->addr = addr;
				args->s_client = &(sock_dict[index_value].s_client);
				args->src_port = ntohs(addr.sin_port);
				args->s_server = sock;
				args->activeClient = activeClient;

				pthread_t thread;
				int thread_create_result = pthread_create(&thread, NULL, listen_thread, args);
				if(thread_create_result != 0)
				{
					perror("pthread_create");
					free(args);
					close(s_client);
					sock_dict[index_value].s_client = -1;
					sock_dict_size--;
				}
				else
					pthread_detach(thread);
			}
			else
				sendto(sock_dict[index_value].s_client, buffer, bytes_received, 0, (struct sockaddr *)&forward_addr, sizeof(forward_addr));
		}
		else
			printf("Invalid address: %s\n", inet_ntoa(addr.sin_addr));
		
		pthread_mutex_unlock(&lock);
	}
	return 0;
}
