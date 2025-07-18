+++
title = "I Built a Web Server in C"
date = "2025-07-11T02:39:59Z"
author = "iniyan"
authorTwitter = ""
cover = ""
tags = ["code", "networking", "scratch", "c"]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = true
hideComments = false
+++

# What Could Possibly Go Wrong?
{{< image src="https://media1.tenor.com/m/8niR84ZU5G4AAAAC/how-hard.gif" alt="how hard" position="center" style="border-radius: 8px;" >}}

I had a week of free time after my end semester. So instead of grinding games and binge-watching movies or shows, I decided: Why not build a web server?

<!--more-->
At first, I considered writing it in Python — but then I realized chads build hobby projects in C. Who needs comfort when you can wrestle with sockets, pointers, and the majestic absence of real string handling?

This project took me about 3 days of tinkering, trial and error, intense Googling, and threatening ChatGPT. But it’s been one of the most fun things I’ve built in a while.

# The Idea

Web servers — or the whole internet really — work by sending and receiving requests.  
When a browser asks for a page like `/about`, it sends something like:

```

GET /about HTTP/1.1

```

That's the page requested by the browser. I need to parse this request and determine which page to return as a response. In complex servers, this is handled by routing logic, templating engines, middleware, etc. But I'm just implementing a simple **file-based routing**.

So the logic is simple: each route should map to a folder, and that folder should have an `index.html`. If a browser requests `/about`, my server will look for `about/index.html` and serve the requested file with a set of response headers like this:

```

HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 2204
Connection: close

````

Then the browser parses this header and renders it based on the **Content-Type** (MIME type of the file).  
If it weren’t for these headers, then it's just a TCP server and client sending and receiving strings.

# Sockets 101

To get anything working on the internet — whether it's a website, chat app, or a game — it needs sockets.

In my case, I wanted a server that does these three things:

1. Create a socket that listens on a specific port  
2. Accept incoming connections from browsers  
3. Read the request, figure out what's being asked for, and send back the file  

This is the lifecycle of the server:

```c
socket() -> create a socket  
bind() -> assign my socket to a port  
listen() -> wait for incoming connections  
accept() -> accept a connection from the client  
recv() -> read the client's request  
send() -> send back a response  
close() -> close the connection  
````

In C, sockets are handled by a file descriptor. You can simply communicate with internet sockets just like reading and writing to a file — unless you want to hate your life.

Besides these core functions, you also deal with a bunch of structs to handle IP addresses, port numbers, and byte-order conversions. It's like trying to talk to the internet like a caveman — but more fun.
If you want to learn more about networking, check out this [legendary guide](https://beej.us/guide/bgnet0/).

# The Server

So after digging into sockets, I started writing the server.

{{<code language="c" title="main" open="false" >}}
int main(){
	Configs *config =parse_config();
	strcpy(PORT, config->port);
	BACKLOG = config->backlog;
	struct sockaddr_storage their_addr;
	int new_fd;
	socklen_t sin_size;
	int numbytes;
	// socket creation
	int sockfd = setup_socket(PORT);
	if(sockfd==-1){
		fprintf(stderr, "server: failed to bind");
		exit(1);
	}
	if(listen(sockfd, BACKLOG) == -1){
		perror("listen");
		exit(1);
	}
	printf("Waiting for connections...\n");
	// start listening to connections
	while(1){
		sin_size = sizeof their_addr;
		new_fd = accept(sockfd, (struct sockaddr *)&their_addr, &sin_size);
		if(new_fd == -1){
			perror("accept");
			continue;
		}
		ThreadArgs *args = malloc(sizeof(ThreadArgs));
		args->sock_fd = new_fd;
		args->root = config->root;
		inet_ntop(their_addr.ss_family, get_in_addr((struct sockaddr *)&their_addr), args->addr, sizeof(  args->addr )); // convert network address to presentable
		thrd_t t;
		if ( thrd_create(&t, handle_client, args) != thrd_success ){
			perror("Failed to create a thread");
			send_error(new_fd, 500);
			free(args);
			continue;
		}
		thrd_detach(t);
	}
}
{{</code>}}

I start by parsing a JSON file containing configurations for the server like the port, number of backlogs, and the root directory (the directory from which the files are served).
Parsing the JSON files was simpler than I thought — thanks to [cJSON](https://github.com/DaveGamble/cJSON).

Then I create a socket and bind it to the port specified in the config, and set it up for listening mode.

Here's the `setup_socket` function if you want to look at it:

{{<code language="c" title="setup socket" open="false" >}}
int setup_socket(char *PORT){

	int sockfd=-1;
	struct addrinfo hints, *servinfo, *p;
	int yes = 1;
	int rv;
	memset(&hints, 0, sizeof hints);
	hints.ai_family = AF_UNSPEC;
	hints.ai_flags = AI_PASSIVE;
	hints.ai_socktype = SOCK_STREAM;

	if((rv = getaddrinfo(NULL, PORT, &hints, &servinfo)) != 0){
		fprintf(stderr, "getaddrinfo : %s\n", gai_strerror(rv));
		return -1;
	}

	for(p=servinfo; p!=NULL; p=p->ai_next){
		if((sockfd = socket(p->ai_family, p->ai_socktype, p->ai_protocol))==-1){
			perror("server: socket");
			continue;
		}
		if(setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(int)) == -1){
			perror("setsockopt");
			continue;
		}
		if((bind(sockfd, p->ai_addr, p->ai_addrlen)) == -1){
			close(sockfd);
			perror("server: bind");
			continue;
		}
		break;
	}
	freeaddrinfo(servinfo);
	return sockfd;
}
{{</code>}}

Then the server loop listens for incoming connections and parses the first line of the request header to determine the requested route, method, and protocol.
A real server would also handle cookies, content-type negotiation, persistent connections, etc.

Each connection is handled in a separate thread — and that’s where `handle_client` comes in.

{{<code language="c" title="handle_client" open="false" >}}
int handle_client(void *arg){
	ThreadArgs *args = (ThreadArgs *)arg;
	char *root = args->root;
	char *request = get_request(args->sock_fd);
	if(!request){
		fprintf(stderr, "[%s] Failed to get request\n", args->addr);
		send_error(args->sock_fd, 400);
		close(args->sock_fd);
		free(args);
		return -1;
	}
	// parse the request to determine the html file
	HeaderData *parsed_request = parse_request(request);
	if(!parsed_request){
		fprintf(stderr, "[%s] Failed to parse request\n", args->addr);
		send_error(args->sock_fd, 400);
		close(args->sock_fd);
		free(args);
		return -1;
	}
	char header[256];
	printf("[ %s ] --> %s %s %s\n", args->addr, parsed_request->method,  parsed_request->path, parsed_request->protocol);
	// setup the response header
	FileData *response_header_data = parse_file(parsed_request->path, root);
	if(!response_header_data){
		send_error(args->sock_fd, 404);
		close(args->sock_fd);
		free(args);
		free(parsed_request);
		return -1;
	}
	FILE *template = response_header_data->fd;
	send_response(args->sock_fd, 200, response_header_data->content_type, response_header_data->content_length);
	render_html(args->sock_fd, template);
	shutdown(args->sock_fd, SHUT_WR);
	fclose(template);
	close(args->sock_fd);
	free(parsed_request);
	free(response_header_data);
	free(args);
	return 0;
}
{{</code>}}

The `handle_client` function reads the request, parses it, and serves the corresponding file.
If the file doesn't exist, it sends a 404.

This is where I spent more than 2 hours debugging a segmentation fault — turns out I was trying to close a socket that was already closed 😵‍💫

{{< image src="https://media1.tenor.com/m/rEd35Rfq3m4AAAAd/cat-work-in-progress.gif" alt="how hard" position="center" style="border-radius: 8px;" >}}

# The Router

The routing logic is handled by the `parse_file` function. It takes the path and root directory and tries to find the corresponding file.

{{<code language="c" title="parse_file" open="false" >}}
FileData *parse_file(char *file_name, char *root) {
	if(strstr(file_name, "..")){
		return NULL;
	}
	size_t path_len = strlen(root) + strlen(file_name) + strlen("/index.html") + 1;
	char *file_path = malloc(path_len);
	if (!file_path) return NULL;

	snprintf(file_path, path_len, "%s%s", root, file_name);
	FILE *fd = fopen(file_path, "r");
	char *ext = strrchr(file_name, '.');
	if (!ext) {
		snprintf(file_path, path_len, "%s%s/index.html", root, file_name);
		fd = fopen(file_path, "r");
		if (!fd) {
			free(file_path);
			return NULL;
		}
	}
	ext = strrchr(file_path, '.');
	printf("Serving: %s  ext: %s\n", file_path, ext);
	FileData *result = malloc(sizeof(FileData));
	if (!result) {
		fclose(fd);
		free(file_path);
		return NULL;
	}

	fseek(fd, 0, SEEK_END);
	result->content_length = ftell(fd);
	fseek(fd, 0, SEEK_SET);
	result->fd = fd;

	char *mime;
	if (!ext) mime = "application/octet-stream";
	else if (strcmp(ext, ".html") == 0) mime = "text/html";
	else if (strcmp(ext, ".css") == 0) mime = "text/css";
	else if (strcmp(ext, ".js") == 0) mime = "application/javascript";
	else if (strcmp(ext, ".jpg") == 0 || strcmp(ext, ".jpeg") == 0) mime = "image/jpeg";
	else if (strcmp(ext, ".png") == 0) mime = "image/png";
	else if (strcmp(ext, ".webp") == 0) mime = "image/webp";
	else if (strcmp(ext, ".gif") == 0) mime = "image/gif";
	else mime = "application/octet-stream";

	result->content_type = mime;
	free(file_path);
	return result;
}
{{</code>}}

This function also attempts to prevent directory traversal attacks, though it could use some more rigorous checks.
It also determines the MIME type based on the file extension — essential for proper rendering.
Sometimes, browsers try to guess the MIME type if it's missing from the response header.

# Conclusion

I tested the server with my Hugo blog by pointing the root directory to `./public` — it works great!
But it’s definitely **not secure** for deployment.

The full code can be found here:
👉 [github.com/in1yan/pop-corn-server](https://github.com/in1yan/pop-corn-server)

{{< image src="https://s14.gifyu.com/images/bKlzy.gif" alt="done" position="center" style="border-radius: 8px;" >}}


