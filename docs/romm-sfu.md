IP Leakage: 
WebRTC inherently uses STUN/ICE to discover real IP addresses. Using host networking makes your host's public and private IPs directly visible to clients, which is a privacy concern for some users. 

Mediasoup-Specific Considerations
Mediasoup is designed to be a "minimalist" media handler and does not include built-in signaling security or room management. 

Signaling is Your Responsibility: 
Mediasoup does not secure your signaling (WebSocket/HTTP). You must ensure your signaling server uses WSS/HTTPS. If this is unencrypted, an attacker can hijack sessions or perform man-in-the-middle (MITM) attacks.

Media is Secure: 
The actual video, audio, and data channels in mediasoup are protected by DTLS and SRTP, providing strong encryption and replay protection for the media itself.
    
Input Validation: 
Ensure your backend strictly validates all data coming through the data channels. If an attacker can send malicious game state or control signals, they could potentially crash the emulator or exploit vulnerabilities in the emulator's core. 

Optimize Worker Cores: 
Run one mediasoup worker per CPU core to prevent worker-level bottlenecks from affecting game input data channels.

index.js IS HARD CODED TO 3001 BY DEFAULT
This is preferred because romm uses

notes; environment variables for documentation
index.sj
141-167 - pass env variables for:
listening IP (default 0.0.0.0)
Announced IP ()   COMPATIBLE WITH URLS -- Support Split Horizon networking configuration
iceServers: 
    urls: "stun:turn.technicallycomputers.ca:3478",
    username: "emulatorjs",
    credential: "rCGKgDisoVJcdFRhltm3",
    
    urls: "turn:turn.technicallycomputers.ca:3478",
    username: "emulatorjs",
    credential: "rCGKgDisoVJcdFRhltm3",
    
      // IMPORTANT: We do not currently have explicit client->server signaling
      // to close old producers when the client calls producer.close().
      // If the host re-produces (e.g. after pause/resume), the SFU can end up
      // with multiple server-side producers of the same kind for the same
      // socket, where the older one no longer receives packets.
      // Rejoining clients can then consume the stale producer and see
      // videoWidth/videoHeight remain 0.
      //
      // To keep behavior deterministic: enforce at most one producer per kind
      // per socket by closing/removing any existing same-kind producers here.
      NEED TO REVIEW -- Issue observed in very early version of index.js
      Lines 242-273
      
  // Netplay system messages: host pause/resume notifications.
  // These are simple broadcasts so spectators get an explicit UI cue.
    Lines 673-740
    Can we expand on this to address the issues in the block referenced above?
    
    
documentation
Security

### UserId Spoofing Protection

Persistent userid values preserve consistent player assignment in game rooms, and security features server side protect against spoofing userid.  

Userid is a unique identifier that is generated when the player initiates a connection with the SFU server, and is then stored locally client side.  EmulatorJS maps this id to the playerid, which identifies which character they control with their inputs.  Data packets for input include a userid to identify the player.  P2P does not offer input security, and theoretically a userid can be spoofed to manipulate other player characters.  The SFU server maps Userid to the data channel, and thus will drop packets with a spoofed userid.  Because Sync & Rollback game mode requires deeper synchronicity and management of inputs by utilizing the SFU relay, and because of it's competitive nature, relay channels are enforced for input streams.

Bound userid to socketid server side and made it immutable per connection.  Clients cannot 'switch' userids by sending arbitrary packets.  Clients cannot spoof a userid when negotiating a connection in order to hijack use of their socket, connection is only allowed if userid isn't already connected.  If a user is connected to a data channel, and tries to send an invalid userid, the data packet is dropped, and they are sent an error "userid mismatch for this connection".



### Token Based Authorization

New python endpoint in romm/backend/endpoints/sfu.py (/api/sfu/ endpoint was added in /backend/handler/main.py)
* Mints an HS 256 JWT! 
* SFU_TOKEN_ISSUER = "romm:sfu" 
* SFU_JTI_REDIS_KEY_PREFIX = "sfu:auth:jti"

* SFU server enforces strict authentiction via token verification.  EmulatorJS calls the romm api for an expiring single use token, romm uses the romm_auth_secret_key to sign the token.  The token authenticates the user as a member of the romm site, and grants access to the SFU server for that session.  The session ends if the user has been disconnected from netplay for more than 30 seconds, and accessing netplay features again requires fetching another token.

* Romm API fetches netplayUsername, netplay_username, settings.netplayUsername or settings.netplay_username -- after authentication we can authorize also, and associate with a persistent netplay user name for the account.  This will be cleaned up but acts as a placeholder for now to build persistent netplay identities for romm users later.


