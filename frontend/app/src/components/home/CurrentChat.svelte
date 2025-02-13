<script lang="ts">
    import CurrentChatHeader from "./CurrentChatHeader.svelte";
    import CurrentChatMessages from "./CurrentChatMessages.svelte";
    import Footer from "./Footer.svelte";
    import { closeNotificationsForChat } from "../../utils/notifications";
    import { getContext, onMount, tick } from "svelte";
    import {
        type ChatEvent,
        type ChatSummary,
        type EnhancedReplyContext,
        type EventWrapper,
        type Mention,
        type Message,
        type MessageContent,
        type OpenChat,
        type FilteredProposals,
        type User,
        type ChatIdentifier,
        chatIdentifiersEqual,
        type MultiUserChat,
        CommunityMap,
        type CommunitySummary,
        ICP_SYMBOL,
    } from "openchat-client";
    import PollBuilder from "./PollBuilder.svelte";
    import CryptoTransferBuilder from "./CryptoTransferBuilder.svelte";
    import CurrentChatSearchHeader from "./CurrentChatSearchHeader.svelte";
    import GiphySelector from "./GiphySelector.svelte";
    import MemeBuilder from "./MemeBuilder.svelte";
    import { messageToForwardStore } from "../../stores/messageToForward";
    import { toastStore } from "../../stores/toast";
    import ImportToCommunity from "./communities/Import.svelte";
    import { randomSentence } from "../../utils/randomMsg";
    import { framed } from "../../stores/xframe";
    import { rightPanelHistory } from "../../stores/rightPanel";
    import AcceptRulesModal from "./AcceptRulesModal.svelte";

    type ConfirmedActionEvent =
        | ConfirmedSendMessage
        | ConfirmedForwardMessage
        | ConfirmedRetrySendMessage;

    type ConfirmedSendMessage = {
        kind: "send_message";
        textContent: string | undefined;
        mentioned: User[];
        fileToAttach: MessageContent | undefined;
    };

    type ConfirmedForwardMessage = {
        kind: "forward_message";
        msg: Message;
    };

    type ConfirmedRetrySendMessage = {
        kind: "retry_send_message";
        event: EventWrapper<Message>;
    };

    export let joining: MultiUserChat | undefined;
    export let chat: ChatSummary;
    export let currentChatMessages: CurrentChatMessages | undefined;
    export let events: EventWrapper<ChatEvent>[];
    export let filteredProposals: FilteredProposals | undefined;

    const client = getContext<OpenChat>("client");
    const user = client.user;

    let previousChatId: ChatIdentifier | undefined = undefined;
    let unreadMessages = 0;
    let firstUnreadMention: Mention | undefined;
    let creatingPoll = false;
    let creatingCryptoTransfer: { ledger: string; amount: bigint } | undefined = undefined;
    let selectingGif = false;
    let buildingMeme = false;
    let pollBuilder: PollBuilder;
    let giphySelector: GiphySelector;
    let memeBuilder: MemeBuilder;
    let showSearchHeader = false;
    let searchTerm = "";
    let importToCommunities: CommunityMap<CommunitySummary> | undefined;
    let showAcceptRulesModal = false;
    let sendMessageContext: ConfirmedActionEvent | undefined = undefined;

    $: currentChatTextContent = client.currentChatTextContent;
    $: currentChatReplyingTo = client.currentChatReplyingTo;
    $: currentChatPinnedMessages = client.currentChatPinnedMessages;
    $: currentChatFileToAttach = client.currentChatFileToAttach;
    $: currentChatEditingEvent = client.currentChatEditingEvent;
    $: currentChatDraftMessage = client.currentChatDraftMessage;
    $: lastCryptoSent = client.lastCryptoSent;
    $: messagesRead = client.messagesRead;
    $: directlyBlockedUsers = client.blockedUsers;
    $: showFooter = !showSearchHeader && !client.isReadOnly();
    $: blocked = isBlocked(chat, $directlyBlockedUsers);
    $: communities = client.communities;

    $: canSend = client.canSendMessages(chat.id);
    $: preview = client.isPreviewing(chat.id);
    $: canPin = client.canPinMessages(chat.id);
    $: canBlockUser = client.canBlockUsers(chat.id);
    $: canDelete = client.canDeleteOtherUsersMessages(chat.id);
    $: canReplyInThread = client.canReplyInThread(chat.id);
    $: canReact = client.canReactToMessages(chat.id);
    $: canInvite = client.canInviteUsers(chat.id);
    $: readonly = client.isChatReadOnly(chat.id);

    $: {
        if (previousChatId === undefined || !chatIdentifiersEqual(chat.id, previousChatId)) {
            previousChatId = chat.id;
            showSearchHeader = false;
            unreadMessages = getUnreadMessageCount(chat);
            firstUnreadMention = client.getFirstUnreadMention(chat);

            tick().then(() => {
                if ($messageToForwardStore !== undefined) {
                    forwardMessage($messageToForwardStore);
                    messageToForwardStore.set(undefined);
                }
            });
        }
    }

    onMount(() => {
        return messagesRead.subscribe(() => {
            unreadMessages = getUnreadMessageCount(chat);
            firstUnreadMention = client.getFirstUnreadMention(chat);
        });
    });

    function importToCommunity() {
        importToCommunities = $communities.filter((c) => c.membership.role === "owner");
        if (importToCommunities.size === 0) {
            toastStore.showFailureToast("communities.noOwned");
            importToCommunities = undefined;
        } else {
            rightPanelHistory.set([]);
        }
    }

    function getUnreadMessageCount(chat: ChatSummary): number {
        if (client.isPreviewing(chat.id)) return 0;

        return messagesRead.unreadMessageCount(chat.id, chat.latestMessage?.event.messageIndex);
    }

    function onWindowFocus() {
        closeNotificationsForChat(chat.id);
    }

    function onMarkAllRead() {
        client.markAllRead(chat);
    }

    function createPoll() {
        if (!client.canCreatePolls(chat.id)) return;

        if (pollBuilder !== undefined) {
            pollBuilder.resetPoll();
        }
        creatingPoll = true;
    }

    function tokenTransfer(ev: CustomEvent<{ ledger: string; amount: bigint } | undefined>) {
        creatingCryptoTransfer = ev.detail ?? {
            ledger: $lastCryptoSent ?? client.ledgerCanisterId(ICP_SYMBOL),
            amount: BigInt(0),
        };
    }

    function fileSelected(ev: CustomEvent<MessageContent>) {
        currentChatDraftMessage.setAttachment(chat.id, ev.detail);
    }

    function attachGif(ev: CustomEvent<string>) {
        selectingGif = true;
        if (giphySelector !== undefined) {
            giphySelector.reset(ev.detail);
        }
    }

    function makeMeme() {
        buildingMeme = true;
        if (memeBuilder !== undefined) {
            memeBuilder.reset();
        }
    }

    function replyTo(ev: CustomEvent<EnhancedReplyContext>) {
        showSearchHeader = false;
        currentChatDraftMessage.setReplyingTo(chat.id, ev.detail);
    }

    function searchChat(ev: CustomEvent<string>) {
        showSearchHeader = true;
        searchTerm = ev.detail;
    }

    function createTestMessages(ev: CustomEvent<number>): void {
        if (process.env.NODE_ENV === "production") return;

        function send(n: number) {
            if (n === ev.detail) return;

            sendMessageWithAttachment(randomSentence(), [], undefined);

            window.setTimeout(() => send(n + 1), 500);
        }

        send(0);
    }

    function sendMessage(ev: CustomEvent<[string | undefined, User[]]>) {
        if (!canSend) return;
        let [text, mentioned] = ev.detail;
        if ($currentChatEditingEvent !== undefined) {
            client
                .editMessageWithAttachment(
                    chat.id,
                    text,
                    $currentChatFileToAttach,
                    $currentChatEditingEvent
                )
                .then((success) => {
                    if (!success) {
                        toastStore.showFailureToast("errorEditingMessage");
                    }
                });
        } else {
            sendMessageWithAttachment(text, mentioned, $currentChatFileToAttach);
        }
    }

    function sendMessageWithAttachment(
        textContent: string | undefined,
        mentioned: User[],
        fileToAttach: MessageContent | undefined
    ) {
        if (client.rulesNeedAccepting()) {
            showAcceptRulesModal = true;
            sendMessageContext = {
                kind: "send_message",
                textContent,
                mentioned,
                fileToAttach,
            };
        } else {
            client.sendMessageWithAttachment(
                chat.id,
                events,
                textContent,
                mentioned,
                fileToAttach,
                $currentChatReplyingTo,
                undefined,
                undefined,
                undefined
            );
        }
    }

    export function sendMessageWithContent(ev: CustomEvent<[MessageContent, string | undefined]>) {
        sendMessageWithAttachment(ev.detail[1], [], ev.detail[0]);
    }

    function forwardMessage(msg: Message) {
        if (!canSend || !client.canForward(msg.content)) return;

        if (client.rulesNeedAccepting()) {
            showAcceptRulesModal = true;
            sendMessageContext = {
                kind: "forward_message",
                msg,
            };
        } else {
            client.forwardMessage(chat.id, msg, undefined, undefined, undefined);
        }
    }

    function retrySend(ev: CustomEvent<EventWrapper<Message>>): void {
        if (client.rulesNeedAccepting()) {
            showAcceptRulesModal = true;
            sendMessageContext = {
                kind: "retry_send_message",
                event: ev.detail,
            };
        } else {
            client.retrySendMessage(chat.id, ev.detail, events, undefined, undefined, undefined);
        }
    }

    function setTextContent(ev: CustomEvent<string | undefined>): void {
        currentChatDraftMessage.setTextContent(chat.id, ev.detail);
    }

    function isBlocked(chatSummary: ChatSummary, blockedUsers: Set<string>): boolean {
        return chatSummary.kind === "direct_chat" && blockedUsers.has(chatSummary.them.userId);
    }

    function defaultCryptoTransferReceiver(): string | undefined {
        return $currentChatReplyingTo?.sender?.userId;
    }

    function onAcceptRules(
        accepted: boolean,
        chatRulesVersion: number | undefined,
        communityRulesVersion: number | undefined
    ) {
        if (sendMessageContext === undefined) {
            showAcceptRulesModal = false;
            return;
        }

        if (accepted) {
            switch (sendMessageContext.kind) {
                case "send_message": {
                    client.sendMessageWithAttachment(
                        chat.id,
                        events,
                        sendMessageContext.textContent,
                        sendMessageContext.mentioned,
                        sendMessageContext.fileToAttach,
                        $currentChatReplyingTo,
                        undefined,
                        chatRulesVersion,
                        communityRulesVersion
                    );
                    break;
                }
                case "forward_message": {
                    client.forwardMessage(
                        chat.id,
                        sendMessageContext.msg,
                        undefined,
                        chatRulesVersion,
                        communityRulesVersion
                    );
                    break;
                }
                case "retry_send_message": {
                    client.retrySendMessage(
                        chat.id,
                        sendMessageContext.event,
                        events,
                        undefined,
                        chatRulesVersion,
                        communityRulesVersion
                    );
                    break;
                }
            }
        } else {
            switch (sendMessageContext.kind) {
                case "send_message": {
                    currentChatDraftMessage.setTextContent(chat.id, sendMessageContext.textContent);
                    currentChatDraftMessage.setAttachment(chat.id, sendMessageContext.fileToAttach);
                    break;
                }
            }
        }

        sendMessageContext = undefined;
        showAcceptRulesModal = false;
    }
</script>

<svelte:window on:focus={onWindowFocus} />

{#if showAcceptRulesModal}
    <AcceptRulesModal action={onAcceptRules} />
{/if}

{#if importToCommunities !== undefined}
    <ImportToCommunity
        on:successfulImport
        groupId={chat.id}
        on:cancel={() => (importToCommunities = undefined)}
        ownedCommunities={importToCommunities} />
{/if}

<PollBuilder
    bind:this={pollBuilder}
    on:sendPoll={sendMessageWithContent}
    bind:open={creatingPoll} />

{#if creatingCryptoTransfer !== undefined}
    <CryptoTransferBuilder
        {chat}
        ledger={creatingCryptoTransfer.ledger}
        draftAmount={creatingCryptoTransfer.amount}
        defaultReceiver={defaultCryptoTransferReceiver()}
        on:sendTransfer={sendMessageWithContent}
        on:upgrade
        on:close={() => (creatingCryptoTransfer = undefined)} />
{/if}

<GiphySelector
    bind:this={giphySelector}
    bind:open={selectingGif}
    on:sendGiphy={sendMessageWithContent} />

<MemeBuilder
    bind:this={memeBuilder}
    bind:open={buildingMeme}
    on:sendMeme={sendMessageWithContent} />

<div class="wrapper">
    {#if showSearchHeader}
        <CurrentChatSearchHeader
            {chat}
            bind:searchTerm
            on:goToMessageIndex
            on:close={() => (showSearchHeader = false)} />
    {:else if !$framed}
        <CurrentChatHeader
            on:clearSelection
            on:markAllRead={onMarkAllRead}
            on:toggleMuteNotifications
            on:showInviteGroupUsers
            on:showProposalFilters
            on:showGroupMembers
            on:leaveGroup
            on:upgrade
            on:createPoll={createPoll}
            on:searchChat={searchChat}
            on:convertGroupToCommunity
            on:importToCommunity={importToCommunity}
            {blocked}
            {readonly}
            {unreadMessages}
            selectedChatSummary={chat}
            hasPinned={$currentChatPinnedMessages.size > 0} />
    {/if}
    <CurrentChatMessages
        bind:this={currentChatMessages}
        on:replyPrivatelyTo
        on:replyTo={replyTo}
        on:chatWith
        on:upgrade
        on:forward
        on:retrySend={retrySend}
        {chat}
        {events}
        {filteredProposals}
        {canPin}
        {canBlockUser}
        {canDelete}
        {canReplyInThread}
        {canSend}
        {canReact}
        {canInvite}
        {readonly}
        {firstUnreadMention}
        footer={showFooter}
        {unreadMessages} />
    {#if showFooter}
        <Footer
            {chat}
            fileToAttach={$currentChatFileToAttach}
            editingEvent={$currentChatEditingEvent}
            replyingTo={$currentChatReplyingTo}
            textContent={$currentChatTextContent}
            {user}
            mode={"message"}
            {joining}
            {preview}
            {blocked}
            on:joinGroup
            on:upgrade
            on:cancelReply={() => currentChatDraftMessage.setReplyingTo(chat.id, undefined)}
            on:clearAttachment={() => currentChatDraftMessage.setAttachment(chat.id, undefined)}
            on:cancelEditEvent={() => currentChatDraftMessage.clear(chat.id)}
            on:setTextContent={setTextContent}
            on:startTyping={() => client.startTyping(chat, user.userId)}
            on:stopTyping={() => client.stopTyping(chat, user.userId)}
            on:fileSelected={fileSelected}
            on:audioCaptured={fileSelected}
            on:sendMessage={sendMessage}
            on:createTestMessages={createTestMessages}
            on:attachGif={attachGif}
            on:makeMeme={makeMeme}
            on:tokenTransfer={tokenTransfer}
            on:searchChat={searchChat}
            on:createPoll={createPoll} />
    {/if}
</div>

<style lang="scss">
    .wrapper {
        display: flex;
        flex-direction: column;
        justify-content: space-between;
        height: 100%;
        position: relative;
    }
</style>
