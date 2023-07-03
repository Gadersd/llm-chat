<script lang="ts">
    import { onMount } from 'svelte';
    import { tick } from 'svelte';

    enum Sender {
        User, 
        Assistant, 
        System, 
    };

    type Message = {
        id: number,
        sender: Sender, 
        text: string
    };

    let messageContainer: HTMLDivElement; // Will hold the reference to the div that contains messages

    let ip: string  = '';
    let isIPConfirmed = false;
    let isIPIncorrect = false;

    let newMessage = '';
    let messages: Message[] = [
        { id: 0, sender: Sender.Assistant, text: 'Hello, how can I assist you today?' },
    ];
    let isLoading = false;
    let ws: WebSocket;

    // Automatically scroll to bottom on message update
    $: scrollToEnd(messageContainer);

    // Scroll to the last message
    async function scrollToEnd(container) {
        if (container) {
            await tick();
            container.scrollTo(0, container.scrollHeight);
        }
    }

    const addMessage = (sender: Sender, text: string) => {
        const last_id = messages.length > 0 ? messages[messages.length - 1].id : 0;
        messages.push( { id: last_id + 1, sender: sender, text: text } );
        messages = messages;
    };

    function connectWebSocket(url, timeout) {
        timeout = timeout || 2000;
        return new Promise(function(resolve, reject) {
            // Create WebSocket connection.
            const socket = new WebSocket(url);

            const timer = setTimeout(function() {
                reject(new Error("webSocket timeout"));
                done();
                socket.close();
            }, timeout);

            function done() {
                // cleanup all state here
                clearTimeout(timer);
                socket.removeEventListener('error', error);
            }

            function error(e) {
                reject(e);
                done();
            }

            socket.addEventListener('open', function() {
                resolve(socket);
                done();
            });
            socket.addEventListener('error', error);
        });
    }

    const connect = (): WebSocket => {
        let buffer = '';
        const extraneous = '\nUser';  // define the extraneous token here

        const stop_token_stream = () => {
            isLoading = false;
            ws.close();
        };

        const add_token = (token: string) => {
            messages[messages.length - 1].text += token;
        };

        const handleToken = token => {
            if (token !== "") {
                buffer += token;
                if (definitelyNoExtraneousToken(buffer, extraneous)) {
                    add_token(token);
                    buffer = '';
                } else if (definitelyHasExtraneousToken(buffer, extraneous)) {
                    // Exit early to prevent assistant playing the part of the user
                    let clipped = removeAllTracesExtraneousToken(buffer, extraneous);
                    if(clipped !== ""){
                        add_token(clipped);
                    }
                    stop_token_stream();
                }
            } else {
                // Received end token
                let clipped = removeAllTracesExtraneousToken(buffer, extraneous);
                if(clipped !== ''){
                    add_token(clipped);
                }
                stop_token_stream();
            }
        };

        let ws: WebSocket;
        isLoading = true;
        connectWebSocket(`wss://${ip}:8765/chat`, 3000).then((socket) => {
            ws = socket;

            ws.onmessage = (event) => {
                const response = event.data;
                handleToken(response);
            };

            ws.onerror = (err) => {
                addMessage(Sender.System, 'Lost connection.');
            };

            ws.onclose = () => {
                isLoading = false;
            };

            console.log('Connection to assistant server is open!');

            // Add latest user prompt and skeleton assistant message
            addMessage(Sender.User, newMessage);
            addMessage(Sender.Assistant, '');

            newMessage = '';

            let full_prompt = format_conversation(messages);
            //console.log('Full prompt: ', full_prompt);
            ws.send(full_prompt);
        }).catch((err) => {
            addMessage(Sender.System, 'Could not connect to assistant server.');
            isLoading = false;
        });

        return ws;
    };

    function definitelyNoExtraneousToken(resp: string, extraneous: string): boolean {
        return !resp.includes(extraneous) && ![...Array(extraneous.length).keys()].some(i => resp.endsWith(extraneous.slice(0, i + 1)));
    }

    function definitelyHasExtraneousToken(resp: string, extraneous: string): boolean {
        return resp.includes(extraneous);
    }

    function removeAllTracesExtraneousToken(resp: string, extraneous: string): string {
        resp = resp.split(extraneous)[0]; // Remove everything after extraneous
        for(let i = extraneous.length; i > 0; i--){
            if(resp.endsWith(extraneous.slice(0, i))){
                return resp.slice(0, -i);
            }
        }
        return resp;
    }

    const format_conversation = (messages: Message[]): string => {
        const role = 'You are a helpful assistant.'
        const convo = messages
            .filter(m => m.sender !== Sender.System)
            .map(m => `${m.sender === Sender.User ? 'User' : 'Assistant'}: ${m.text}`)
            .join('\n');

        return `${role}\n\n${convo}`;
    };

    const sendUserPrompt = () => {
        ws = connect();
    };

    // RegExp for a simple validation of an IPv4
    const ipPattern = 
        /^(\d|[1-9]\d|1\d\d|2([0-4]\d|5[0-5]))\.(\d|[1-9]\d|1\d\d|2([0-4]\d|5[0-5]))\.(\d|[1-9]\d|1\d\d|2([0-4]\d|5[0-5]))\.(\d|[1-9]\d|1\d\d|2([0-4]\d|5[0-5]))$/;

    // function to confirm the IP input
    function confirmIP(testIP: string) {
        if (ipPattern.test(testIP)) {
            isIPConfirmed = true;
            isIPIncorrect = false;
        } else {
            isIPIncorrect = true;
        }
    }
  </script>
  
  <style lang="postcss">
    .message-text {
        white-space: pre-line;
    }

    .sender-container {
        flex: 0 0 60px;
    }

    .message-container {
        flex: 1 1 auto;
    }
  </style>

{#if !isIPConfirmed}
    <p class="text-lg text-gray-700">Please enter the IP address of the assistant server:</p>
    <input class="border-gray-300 shadow-sm rounded-md px-3 py-2 bg-white text-gray-700" bind:value={ip} placeholder="Enter IP here" on:keydown={(e) => e.key === 'Enter' && confirmIP(ip)}/>
    <button class="bg-blue-500 text-white rounded-md px-3 py-2 mt-2 hover:bg-blue-700 transition-colors" on:click={() => confirmIP(ip)}>Confirm</button>
    {#if isIPIncorrect}
        <p class="text-lg text-red-500 mt-2">The IP address you entered is invalid. Please try again.</p>
    {/if}
{:else}
    <div class="h-[80vh] overflow-auto bg-white p-6 rounded-md shadow-md space-y-4" bind:this={messageContainer}>
        {#each messages as message (message.id)}
            <div class={`p-4 rounded-md shadow-sm ${message.sender === Sender.User ? 'bg-blue-200 text-blue-800' : message.sender === Sender.Assistant ? 'bg-green-200 text-green-800' : 'bg-gray-200 text-gray-800'}`}>
                <div class="flex">
                    <div class="sender-container">
                        <p>
                            {#if message.sender === Sender.User}
                                You: 
                            {:else if message.sender === Sender.Assistant}
                                Bot: 
                            {:else}
                                System: 
                            {/if}
                        </p>
                    </div>
                    <div class='message-container'>
                        <p class="message-text">
                            {message.text}
                        </p>
                    </div>
                </div>
            </div>
        {/each}
    </div>
    <div class="flex items-center justify-between mt-4">
        <textarea class="flex-grow mr-2 border-gray-300 shadow-sm rounded-md px-3 py-2 bg-white text-gray-700" bind:value={newMessage} placeholder="Your message..." on:keydown={(e) => { e.key === 'Enter' && !e.shiftKey && !isLoading && sendUserPrompt(); e.key === 'Enter' && !e.shiftKey && e.preventDefault(); }} />
        <button class="bg-blue-500 text-white rounded px-4 py-2 {isLoading ? 'opacity-50 cursor-not-allowed' : ''}" on:click={() => !isLoading && sendUserPrompt()} disabled={isLoading}>Send</button>
    </div>
{/if}

