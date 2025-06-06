<script setup>
import { ref, watch } from 'vue';
import axios from 'axios';
import { validateFen } from 'fentastic';
import MarkdownIt from 'markdown-it';

const server_url = import.meta.env.BASE_URL + 'backend'
const starting_fen = 'rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1'
//emits
const emit = defineEmits(['loadingChat']);

// Props
const props = defineProps({
    fen: {
        type: String,
        required: true,
    },
});
// Markdown
const md = new MarkdownIt();


// Reactive state
const userInput = ref('');
const messages = ref([]);
const loading = ref(false)
const toAnalyse = ref(true);
// Methods
async function sendMessage() {
    if (userInput.value.trim() === '') return; //No empty messages

    messages.value.push({ role: 'user', content: userInput.value });
    userInput.value = ''; // Clear input field
    loading.value = true;
    emit('loadingChat', true)

    try {
        const response = await axios.post(server_url + '/response',
            JSON.stringify(messages.value),
            {
                headers: {
                    'Content-Type': 'application/json'
                }
            }
        );
        messages.value = response.data
    } catch (error) {
        console.error('Error querying the LLM:', error);
        messages.value.push({ role: 'assistant', content: 'Error: unable to fetch response.' });
    } finally {
        loading.value = false;
        emit('loadingChat', false)  // Stop showing "thinking"
    }
}

async function startAnalysis() {
    toAnalyse.value = false;
    messages.value.length = 0; // A new analysis only has messages pertaining to that analysis

    const fenToAnalyse = validateFen(props.fen.trim()).valid ? props.fen.trim() : starting_fen
    if (validateFen(fenToAnalyse).valid) {
        loading.value = true;
        emit('loadingChat', true)
        
        try {
            const analysis = await axios.post(server_url + '/analysis',
                JSON.stringify({ fen: props.fen }),
                {
                    headers: {
                        'Content-Type': 'application/json'
                    }
                }
            );
            if (Array.isArray(analysis.data)) {
                messages.value = analysis.data;
            } else {
                console.error('Unexpected response format:', analysis.data);
            }
        } catch (error) {
            console.error('Error querying the LLM:', error);
            messages.value.push({ role: 'assistant', content: 'Error: unable to fetch analysis.' });
        } finally {
            loading.value = false;  
            emit('loadingChat', false)
        }
    }
}

// Watcher
watch(() => props.fen, () => {
    toAnalyse.value = true;
});

function isFirstPrompt(stringToCheck) { //Shitties way to do this, but oh well
    let isFirstPrompt = (stringToCheck.includes("I will explain the board situation:"));
    return isFirstPrompt
}

function renderedMarkdown(content) {
    return md.render(content)
}
</script>


<template>
    <div
        class="container d-flex flex-column justify-content-between overflow-auto p-3 me-0 rounded-bottom rounded-4 w-100 h-100">
        <!-- Chat Messages -->
        <div id="messages" class="flex-item">
            <div v-for="(message, i) in messages" :key="i">
                <div v-if="message.role === 'user' && !isFirstPrompt(message.content)"
                    class="d-flex mb-1 justify-content-end">
                    <div class="p-3 px-4 rounded-4 ms-5" id="usermessage">
                        <div class="text-break text-start justify-content-start message"
                            v-html="renderedMarkdown(message.content)"></div>
                    </div>
                </div>

                <div v-else-if="message.role === 'assistant'" class="p-3 pe-4 rounded-4 me-5">
                    <h6 class="mb-0">AI:</h6>
                    <div class="text-break text-start message" v-html="renderedMarkdown(message.content)"></div>
                </div>
            </div>
            <div v-if="loading" class="thinking-indicator p-3 pe-4 rounded-4 me-5">
                <div class="text-break text-start text-muted">
                    <span class="spinner-border spinner-border-sm" role="status" aria-hidden="true"></span>
                </div>
            </div>
        </div>
        <div v-if="toAnalyse" class="d-flex justify-content-center">
            <button type="button" class="btn btn-sm m-1 text-black custom-bg-primary px-5 py-3 fw-bold" @click="startAnalysis">
                Analyze
            </button>
            <!-- Input Field -->

        </div>
        <input v-model="userInput" v-else @keyup.enter="sendMessage" id="input"
            class="flex-item border rounded px-3 py-2 mt-2 w-100 text-white custom-box" placeholder="Ask Anything!"
            autocomplete="off" />

    </div>
</template>

<style scoped>
.custom-box {
    background-color: #2e2e2e;
    border-color: #2e2e2e !important;
    outline: none;
}
.custom-bg-primary {
    border: 2px solid #aaa23a;
}
.custom-bg-primary:hover {
    border: 2px solid #aaa23a;
    background-color: transparent;
    color: #aaa23a !important;
}
#input::placeholder {
    color: #b2b2b2;
}

#usermessage {
    background: #323232 !important;
}

.message>* {
    margin: 0% !important;
}

.spinner-border,
h6 {
    color: #aaa23a;
    /* Spinner color */
}
</style>
