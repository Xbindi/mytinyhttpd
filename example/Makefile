all: echoserver echoclient UDPserver UDPclient

echoserver: echoserver.cpp
	g++ -W -Wall -o $@ $< 

echoclient: echoclient.cpp
	g++ -W -Wall -o $@ $<

UDPserver: UDPserver.cpp
	g++ -W -Wall -o $@ $< 

UDPclient: UDPclient.cpp
	g++ -W -Wall -o $@ $<
	
clean:
	rm echoserver echoclient UDPserver UDPclient