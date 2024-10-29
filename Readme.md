# Way2Connect: Secure P2P Chat Widget with Video & Voice Calls Code Snippet

Hey there! 👋 We've built this with security and seamless communication in mind, offering peer-to-peer video and voice calls that'll make your users feel like they're in the same room.

## What's in the Box? 🎁

- Real-time chat functionality
- Secure P2P video and voice calls
- File sharing capabilities
- Conversation rating system
- Appointment scheduling with slot management
- Multi-language support


## Tech Stack 🛠️
### Backend: Laravel 10

- Service-Repository pattern for clean, maintainable code
- Event-driven architecture for real-time updates
- Robust error handling and logging


### Frontend: Vue.js 3 with Pinia

- State management with Pinia
- Component-based architecture for reusability
- Real-time updates using WebSockets
- P2P voice and video call using WebRTC


## Backend Showcase

### ConversationRepository

```php
<?php

namespace App\Repositories;

use App\Models\Conversation;
use App\Models\Session;
use Illuminate\Database\Eloquent\Collection;

class ConversationRepository
{
    public function create(array $attributes): Conversation
    {
        return Conversation::create($attributes);
    }

    public function find(int $id): ?Conversation
    {
        return Conversation::find($id);
    }

    public function getMessages(Conversation $conversation, int $limit, int $page = 1): Collection
    {
        return $conversation->messages()
            ->orderBy('created_at', 'desc')
            ->skip(($page - 1) * $limit)
            ->take($limit)
            ->get();
    }

    public function getQueueNumber(int $conversationId): int
    {
        return Conversation::where('status', 'pending')
            ->where('id', '<=', $conversationId)
            ->count();
    }

    public function getReservedSlots(): array
    {
        return Conversation::where('status', 'slot')
            ->pluck('slot_time')
            ->toArray();
    }
}
```

### ConversationService

```php
<?php

namespace App\Services;

use App\Models\Conversation;
use App\Models\Session;
use App\Repositories\ConversationRepository;
use App\Events\Conversations\MessageSent;
use Illuminate\Support\Facades\DB;

class ConversationService
{
    protected $repository;

    public function __construct(ConversationRepository $repository)
    {
        $this->repository = $repository;
    }

    public function create(array $attributes): Conversation
    {
        return DB::transaction(function () use ($attributes) {
            $conversation = $this->repository->create($attributes);
            $this->sendSystemMessage($conversation, tenant()->welcome_message);
            return $conversation;
        });
    }

    public function sendMessage(Conversation $conversation, $sender, string $message): void
    {
        $messageModel = $conversation->messages()->create([
            'sender_type' => get_class($sender),
            'sender_id' => $sender->id,
            'message' => $message,
        ]);

        event(new MessageSent($messageModel));
    }

    public function sendSystemMessage(Conversation $conversation, string $message): void
    {
        $this->sendMessage($conversation, tenant()->user, $message);
    }

    public function getMessages(Conversation $conversation, int $limit, int $page = 1): array
    {
        return $this->repository->getMessages($conversation, $limit, $page)->toArray();
    }

    public function getQueueNumber(Conversation $conversation): int
    {
        return $this->repository->getQueueNumber($conversation->id);
    }

    public function getSlotsConfig(): array
    {
        return json_decode(tenant()->slots, true) ?? [];
    }

    public function getReservedSlots(): array
    {
        return $this->repository->getReservedSlots();
    }
}
```

### Conversation Model

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use App\Events\Conversations\ConversationCreated;
use App\Events\Conversations\ConversationUpdated;

class Conversation extends Model
{
    protected $fillable = ['session_id', 'user_id', 'status', 'slot_time'];

    protected $dispatchesEvents = [
        'created' => ConversationCreated::class,
        'updated' => ConversationUpdated::class,
    ];

    public function session()
    {
        return $this->belongsTo(Session::class);
    }

    public function messages()
    {
        return $this->hasMany(ConversationMessage::class);
    }

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

### ConversationCreated Event

```php
<?php

namespace App\Events\Conversations;

use App\Models\Conversation;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class ConversationCreated implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public $conversation;

    public function __construct(Conversation $conversation)
    {
        $this->conversation = $conversation;
    }

    public function broadcastOn()
    {
        return new Channel('conversations');
    }
}
```

### ConversationUpdated Event

```php
<?php

namespace App\Events\Conversations;

use App\Models\Conversation;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class ConversationUpdated implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public $conversation;

    public function __construct(Conversation $conversation)
    {
        $this->conversation = $conversation;
    }

    public function broadcastOn()
    {
        return new Channel('conversations');
    }
}
```

### MessageSent Event

```php
<?php

namespace App\Events\Conversations;

use App\Models\ConversationMessage;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class MessageSent implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public $message;

    public function __construct(ConversationMessage $message)
    {
        $this->message = $message;
    }

    public function broadcastOn()
    {
        return new Channel('conversation.' . $this->message->conversation_id);
    }
}
```

### ConversationsController

```php
<?php

namespace App\Http\Controllers\API;

use App\Http\Controllers\Controller;
use App\Http\Requests\Conversations\CreateConversationRequest;
use App\Services\ConversationService;
use App\Models\Session;
use App\Models\User;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

class ConversationsController extends Controller
{
    protected $conversationService;

    public function __construct(ConversationService $conversationService)
    {
        $this->conversationService = $conversationService;
    }

    public function create(CreateConversationRequest $request)
    {
        try {
            DB::beginTransaction();

            $session = $this->updateOrCreateSession($request);
            $conversation = $this->conversationService->create(['session_id' => $session->id]);
            $this->conversationService->sendMessage($conversation, $session, $request->validated('message'));

            DB::commit();

            return $this->successResponse($conversation);
        } catch (\Exception $e) {
            DB::rollBack();
            Log::error('Conversation creation failed', ['error' => $e->getMessage()]);
            return $this->errorResponse('Failed to create conversation');
        }
    }

    protected function updateOrCreateSession(CreateConversationRequest $request): Session
    {
        return tap($request->session(), function ($session) use ($request) {
            $session->fill($request->validated());
            $session->save();
        });
    }

    protected function successResponse($conversation)
    {
        $onlineAgents = User::whereIn('id', tenant()->users()->pluck('user_id'))
                            ->where('is_online', true)
                            ->count();

        return response()->json([
            'status' => 'success',
            'data' => [
                'conversation' => $conversation->fresh()->load('session'),
                'queue_number' => $this->conversationService->getQueueNumber($conversation),
                'messages' => $this->conversationService->getMessages($conversation, config('app.pagination_limit')),
                'slots_config' => $this->conversationService->getSlotsConfig(),
                'slots_reserved' => $this->conversationService->getReservedSlots(),
                'online_agents' => $onlineAgents
            ]
        ]);
    }

    protected function errorResponse(string $message)
    {
        return response()->json(['status' => 'error', 'message' => $message], 500);
    }
}
```

## Frontend Showcase

### Conversations Store (Pinia)

```javascript
import { defineStore } from 'pinia'
import { useWidgetStore } from "./WidgetStore";
import { httpServer } from "../core/http-server";
import Messages from "../lang/Messages";

export const useConversationsStore = defineStore('conversations', {
    state: () => ({
        api: '/conversations',
        conversations: [],
        conversation: {},
        slotConfig: [],
        slotReserved: [],
        slot: {},
        messages: [],
        callMessage: null,
        onlineAgents: 0,
        call: null,
        callChatToggle: true,
        loadingLabels: { "accepted": "Initializing The Call...", "preparing": "Almost there...." },
        callDefaultOptions: { audio: true, screen: false, video: false },
        unpluggedDevices: { audio: false, video: false },
        callRemoteOptions: { audio: true, video: false },
        callElapsedTime: 0,
        activeDialog: 'chat',
        socket: null,
        oldTitle: null,
        downloadingFiles: [],
        queueNumber: null,
        loading: false,
        showToast: false,
        headers: useWidgetStore().headers
    }),
    actions: {
        async getList(page) {
            try {
                const { data } = await httpServer.get(this.api, { headers: this.headers, params: { page } });
                if (data.status !== 'success') {
                    throw new Error(data.errors)
                }
                if (data.conversations.data.length !== 0) {
                    this.conversations.push(...data.conversations.data);
                }
                return {
                    status: 'success',
                    page: parseInt(data.conversations.current_page),
                    lastPage: data.conversations.last_page
                }
            } catch (error) {
                return { status: 'error', 'errorMessage': error.general || null };
            }
        },
        async create(params) {
            try {
                const { data } = await httpServer.post(this.api, params, { headers: this.headers });
                if (data.status !== 'success') {
                    return { status: 'error', errors: data.errors };
                }
                this.conversation = data.conversation;
                this.messages = data.messages.data;
                this.queueNumber = data.queue_number;
                this.slotConfig = data.slots_config;
                this.slotReserved = data.slots_reserved;
                this.onlineAgents = data.online_agents;
                return { status: 'success', id: data.conversation.id }
            } catch (error) {
                return { status: 'error', errors: Messages.unexpected };
            }
        },
        async getInformation(conversationId) {
            try {
                const { data } = await httpServer.get(this.api + "/" + conversationId, {
                    headers: this.headers
                });
                if (data.status !== 'success') {
                    throw new Error(data.errors)
                }
                this.conversation = data.conversation;
                this.slotConfig = data.slots_config;
                this.slotReserved = data.slots_reserved;
                this.messages = data.messages.data;
                this.slot = data.slot;
                this.queueNumber = data.queue_number;
                this.onlineAgents = data.online_agents;
                return {
                    status: 'success',
                    page: parseInt(data.messages.current_page),
                    lastPage: data.messages.last_page
                }
            } catch (error) {
                return { status: 'error', 'errorMessage': error.general || null };
            }
        },
        async loadMoreMessages(conversationId, page) {
            try {
                const { data } = await httpServer.get(this.api + "/" + conversationId, {
                    headers: this.headers,
                    params: { page, lastId: this.messages[this.messages.length - 1].id }
                });
                if (data.status !== 'success') {
                    throw new Error(data.errors)
                }
                this.messages.unshift(...data.messages.data);
                return {
                    status: 'success',
                    page: parseInt(data.messages.current_page),
                    lastPage: data.messages.last_page
                }
            } catch (error) {
                return { status: 'error', 'errorMessage': error.general || null };
            }
        },
        async downloadFile(messageId) {
            try {
                const {
                    data,
                    status
                } = await httpServer.get(this.api + "/" + this.conversation.id + "/file/" + messageId, {
                    headers: this.headers,
                    responseType: "blob"
                });
                if (status !== 200) {
                    return { status: 'error', errors: Messages.unexpected };
                }
                return { status: 'success', file: data }
            } catch (error) {
                return { status: 'error', errors: Messages.unexpected };
            }
        },
        async sendMessage(message, resend = false) {
            let messageId;
            try {
                // Push Pending Message Until process finished
                const tempMessage = {
                    is_admin: false,
                    target: { name: this.conversation.session_name },
                    fetching: true,
                    type: "text",
                    message: message,
                    requestData: message,
                }
                if (!resend) {
                    this.messages = [...this.messages, tempMessage];
                }
                //Set Message Id for update it in messages state after request finalized
                messageId = this.messages.length - 1;
                tempMessage.id = messageId;
                const { data } = await httpServer.post(this.api + "/" + this.conversation.id, { message:  tempMessage.message }, { headers: this.headers });
                if (data.status !== 'success') {
                    return { status: 'error', errors: data.errors, messageId: messageId };
                }
                return { status: 'success', message: data.message, messageId: messageId }
            } catch (error) {
                return { status: 'error', errors: Messages.unexpected, messageId: messageId };
            }
        },
        async sendFile(file, resend = false) {
            let messageId;
            try {
                // Push Pending Message Until process finished
                const tempMessage = {
                    is_admin: false,
                    target: { name: this.conversation.session_name },
                    fetching: true,
                    type: "file",
                    message: file.name,
                    requestData: file,
                }
                if (!resend) {
                    this.messages = [...this.messages, tempMessage];
                }
                //Set Message Id for update it in messages state after request finalized
                messageId = this.messages.length - 1;
                const { data } = await httpServer.post(
                    this.api + "/" + this.conversation.id + "/file",
                    { file: file },
                    { headers: { ...this.headers, "Content-Type": "multipart/form-data" } }
                );
                if (data.status !== 'success') {
                    return { status: 'error', errors: data.errors, messageId: messageId };
                }
                return { status: 'success', message: data.message, messageId: messageId }
            } catch (error) {
                return { status: 'error', errors: Messages.unexpected, messageId: messageId };
            }
        },
        async close() {
            try {
                const { data } = await httpServer.get(this.api + "/" + this.conversation.id + "/close", {
                    headers: this.headers,
                });
                if (data.status !== 'success') {
                    throw new Error(data.errors)
                }
                this.conversation = data.conversation;
                return {
                    status: 'success'
                }
            } catch (error) {
                return { status: 'error', 'errorMessage': error.general || null };
            }
        },
        async sendTranscript() {
            try {
                const { data } = await httpServer.get(this.api + "/" + this.conversation.id + "/transcript", {
                    headers: this.headers,
                });
                if (data.status !== 'success') {
                    throw new Error(data.errors)
                }
                return {
                    status: 'success',
                    message: data.message
                }
            } catch (error
```