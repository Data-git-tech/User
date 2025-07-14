# Userhello
DOCS
🎮 Andar Bahar Game Event Documentation

📍 Event List
1. on_start_game
Triggered By: Server


Purpose: Game initialization, sends lobby data (timers, stakes, etc.)


Payload:
{
  "data": {// lobby data}
}



2. on_betting_round_started
Triggered By: Server


Purpose: Notifies players that the betting window has started.


Payload:
{
  "betTimer": 20 // seconds, from lobby data
}



3. on_betting_round_ended
Triggered By: Server


Purpose: Indicates that the betting window is closed.


Payload: {}


4. on_send_bet_data
Triggered By: Client


Purpose: Sends the player's bet amounts on each side.


Payload:
{
  "betOfAndar": 30,
  "betOfBahar": 150
}




5. on_game_finish
Triggered By: Server


Purpose: Sends final result, cards dealt, winning info.


Payload:
{
  "message": "You win 1000 chips.\nThe matching card appeared on Bahar.",
  "showCardDelay": 1.5,
  "matchCard": 34,
  "cards": [/* List of card IDs dealt alternately to Andar/Bahar */],
  "winType": 1,       // 0 = Andar Wins, 1 = Bahar Wins,
  "result": "Bahar",  // Optional: textual result,
  "chipsWin": 1000,
  "chipsLose": 0,
  "chipsBet": 400,
}

✅ Game Requirements
Deck: Standard 52-card deck (no jokers)
Middle Card: Randomly selected card from deck; defines the target rank (e.g., 7♦ → match = any 7)

Game Flow:
Dealer places cards alternately on Andar and Bahar until a card matching the middle card's rank appears

Main Bet Options:
Andar

Bahar


Optional Side Bets:


Predict total cards before match appears (1–5, 6–10, 11–15, 16+)


Payouts vary by card count using GetWinRate(cardUsedCount) logic



🔄 Game State Management
State
Trigger
Next State
Notes
Idle
App Launch
WaitingForStart
Initial state
WaitingForStart
on_start_game
BettingRoundStarted
Lobby and deck initialized
BettingRoundStarted
on_betting_round_started
BettingRoundEnded
Players can now place bets
BettingRoundEnded
on_betting_round_ended
ShowResult
Betting is closed
ShowResult
on_game_finish
WaitingForStart
Show match result, payout, then restart


🎯 Game Overview:
Andar Bahar is a fast-paced Indian card game of chance. The game begins with a middle card drawn from a standard 52-card deck. Players bet on which side — Andar or Bahar — will first receive a card matching the rank of the middle card. Cards are dealt alternately to each side until the match is found.
Players can also place optional side bets to predict how many cards will be dealt before the match, with varying multipliers based on card count.
🎴 No jokers


🎯 Target match based on rank


🃏 Card dealing alternates between Andar and Bahar


💰 Main Bet: Andar or Bahar


🎁 Side Bet: Match card position-based multipliers


FILE CODE :
In-out.js

const lobbyModel = require('../models/lobbyModel');
const userModel = require('../models/userModel');
// const { createDeck, shuffle, getRestartTime } = require('../utils/andarBaharGame');
const { createDeck, shuffle, getWinRate } = require('../utils/in-outRules');

const activeGames = new Map(); // track decks and middle cards by lobbyId
 
const handleAndarBaharStart = async (lobbyId, io) => {
    try {
 
        const lobby = await lobbyModel.getLobbyById(Number(lobbyId));
        if (!lobby) return;
 
        const deck = createDeck();
        shuffle(deck);
        const middleCard = deck.pop();
 
        activeGames.set(String(lobbyId), {
            deck,
            middleCard,
            bets: {},
        });
 
        io.to(String(lobbyId)).emit('on_betting_round_started', {
            betTimer: lobby.betTimer || 20
        });
 
        setTimeout(() => {
            io.to(String(lobbyId)).emit('on_betting_round_ended', {});
        }, (lobby.betTimer || 20) * 1000);
    } catch (error) {
        console.error('Error in startBettingRoundForAndarBahar:', error);
    }
};
 
const handleAndarBaharLogic = async (socket, io, { userId, lobbyId, gameType, data }) => {
    try {
        const userIdNum = Number(userId);
        const lobbyIdStr = String(lobbyId);
        const betOfAndar = isNaN(Number(data?.betOfAndar)) ? 0 : Number(data.betOfAndar);
        const betOfBahar = isNaN(Number(data?.betOfBahar)) ? 0 : Number(data.betOfBahar);
        const totalBet = betOfAndar + betOfBahar;
 
        const gameState = activeGames.get(lobbyIdStr);
        if (!gameState) {
            return socket.emit('on_bet_fail', { message: "Game state not found." });
        }
 
        const user = await userModel.getUserById(userIdNum);
        if (!user || user.chips < totalBet) {
            return socket.emit('on_bet_fail', { message: "Insufficient chips." });
        }
 
        // Deduct chips
        await userModel.removeChipsFromAnder(userIdNum, totalBet);
 
        gameState.bets[userId] = {
            betOfAndar,
            betOfBahar
        };
 
        // Simulate game
        const { deck, middleCard } = gameState;
        const cards = [];
        let matchCard = null;
        let matchIndex = -1;
        let winType = null;
 
        const reverseRankMap = {
            0: '2', 1: '3', 2: '4', 3: '5', 4: '6', 5: '7', 6: '8',
            7: '9', 8: '10', 9: 'J', 10: 'Q', 11: 'K', 12: 'A'
        };
        const suitMap = { 'Clubs': 0, 'Diamonds': 1, 'Hearts': 2, 'Spades': 3 };
        const rankMap = { '2': 0, '3': 1, '4': 2, '5': 3, '6': 4, '7': 5, '8': 6, '9': 7, '10': 8, 'J': 9, 'Q': 10, 'K': 11, 'A': 12 };
        const normalizeRank = (rank) => typeof rank === 'number' ? rankMap[String(rank)] : rankMap[rank];
 
        while (deck.length > 0) {
            const card = deck.pop();
            const suitIndex = suitMap[card.suit];
            const rankIndex = normalizeRank(card.rank);
            const cardId = suitIndex * 13 + rankIndex;
            cards.push(cardId);
 
            if (card.rank === middleCard.rank) {
                matchCard = card.rank;
                matchIndex = cards.length;
                winType = (matchIndex % 2 === 0) ? 0 : 1; // 0 = Andar, 1 = Bahar
                break;
            }
        }
 
        let chipsWin = 0;
        let chipsLose = 0;
 
        if ((winType === 0 && betOfAndar > 0) || (winType === 1 && betOfBahar > 0)) {
            chipsWin += (winType === 0 ? betOfAndar : betOfBahar) * 2;
        } else {
            chipsLose += totalBet;
        }
 
        if (chipsWin > 0) {
            await userModel.addChipsToUser(userIdNum, chipsWin);
        }
 
        const matchRank = reverseRankMap[Number(matchCard)]; // convert 9 → 'J'
 
        const revealedCards = cards.filter(cardId => {
          const rankIndex = cardId % 13;
          const rank = reverseRankMap[rankIndex];
          return rank !== matchRank; // now both are strings
        });
 
        if (gameType === 13) {
            socket.emit('on_game_finish', {
                message: chipsWin > 0 ? `You win ${chipsWin} chips.` : `You lose ${chipsLose} chips.`,
                showCardDelay: 1.5,
                matchCard,
                cards: revealedCards,
                winType,
                result: winType === 0 ? 'Andar' : 'Bahar',
                chipsWin,
                chipsLose,
                chipsBet: totalBet,
            });
        }
 
        const lobby = await lobbyModel.getLobbyById(Number(lobbyId));
        const totalCardsCount = 1 + cards.length;
        const gameRestartTimer = getRestartTime(1.5, totalCardsCount, lobby?.duration || 5);
 
        if (lobby) {
            setTimeout(() => {
                startBettingRoundForAndarBahar(lobbyId, io);
            }, gameRestartTimer * 1000);
        }
 
    } catch (error) {
        console.error('Error in handleAndarBaharLogic:', error);
        socket.emit('on_bet_fail', { message: error.message });
    }
};
 
 
 
module.exports = {
    handleAndarBaharStart,
    handleAndarBaharLogic,
};


in.outRules.js 

const suits = ['Hearts', 'Diamonds', 'Clubs', 'Spades'];
const ranks = [2, 3, 4, 5, 6, 7, 8, 9, 10, 'J', 'Q', 'K', 'A'];
 
function createDeck() {
  const deck = [];
  for (const suit of suits) {
    for (const rank of ranks) {
      deck.push({ suit, rank });
    }
  }
  return deck;
}
 
function shuffle(deck) {
  for (let i = deck.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [deck[i], deck[j]] = [deck[j], deck[i]];
  }
}
 
function getWinRate(cardCount) {
  if (cardCount <= 5) return 5;
  if (cardCount <= 10) return 3;
  if (cardCount <= 15) return 2;
  return 1.5;
}
 
function getRestartTime(showCardDelay, totalCardsCount, restartTime) {
  return (totalCardsCount * showCardDelay) + restartTime;
}
 
module.exports = {
  createDeck,
  shuffle,
  getWinRate,
  getRestartTime
};

SOCKET.JS

   // join gamea
      socket.on('join_lobby', async (message) => {
        console.log("message1",message)

        try {
          const parsed = JSON.parse(message);
          const lobbyId = parsed.data?.lobbyId;
          const userId = parsed.data?.userId;
          

          if (!lobbyId || !userId) {
            socket.emit('join_lobby_error', {
              success: false, 
              message: 'Invalid input id',  
            });
            return; 
          }
          
          // Leave previous rooms before joining a new one
          const rooms = Array.from(socket.rooms);   
          rooms.forEach((room) => {
            if (room !== socket.id) {
              socket.leave(room);
            }
          });
          
          lobbyMap.set(socket.id, {
            userId,
            lobbyId,
          });

          // Join lobby room
          socket.join(lobbyId);

          // Log rooms after joining
          // console.log("Rooms after join:", Array.from(socket.rooms));
          // console.log( 
          //   "All active rooms:",
          //   Array.from(io.sockets.adapter.rooms.keys())
          // );

          io.to(lobbyId).emit('add_in_lobby', {
            message: 'You join the lobby',
          });

          // Call joinLobby function
          await joinLobby(socket, lobbyId, userId);

          // Emit success message 
          // socket.emit("join_lobby_success", {
          //   success: true,
          //   lobbyId,
          //   message: "Successfully joined the lobby",
          // });
        } catch (error) {
          console.error('Error in join_lobby:', error);
          socket.emit('join_lobby_error', {
            success: false,
            message: 'Failed to join lobby',
          });
        }
        console.log("message1", message);
      });
       // game start here
   
       socket.on(
        'start_game_request',
        async ({ lobbyId, success, message, gameLobbies }) => {
          const lobbyIdStr = lobbyId.toString();
          io.to(lobbyIdStr).emit('on_start_game', {   
            success,
            lobbyId: lobbyIdStr,
            message,
            data: gameLobbies, // Provide actual game data here
            channelId: gameLobbies?.channel_id,
          });


          const lobby = await lobbyModel.getLobbyById(Number(lobbyIdStr));
          if (!lobby) {
            console.error(`Lobby ${lobbyId} not found`);
            return;
          }

          console.log( `Lobby ${lobbyId} found:`, lobby);


          // Start the game logic based on the lobby  id=13 of andar bahar gema
           if (lobby?.game_id == 13) {
            await handleAndarBaharStart(lobby?.id, io);
          }

        }
      );

      // in -out
      // handleAndarBaharLogic(socket, io)
       
      // socket.on('on_send_bet_data', ({ userId, lobbyId, gameType, data }) => {
        handleAndarBaharLogic(socket, io, { userId, lobbyId, gameType, data });
      // });
