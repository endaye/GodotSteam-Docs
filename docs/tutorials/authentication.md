# Tutorials - Authentication

In this tutorial we will talk about authenticating users with Steam. Networking is out of scope for this tutorial; you will need to implement your own networking code. You can check out the [lobbies](lobbies.md) and [p2p networking](p2p.md) tutorials for more on that, or even use Godot's high-level networking. [You can read more about the whole authentication process in Steam's documentation page on the subject.](https://partner.steamgames.com/doc/features/auth){ target="_blank" }

The relevant GodotSteam classes and functions for this tutorial are:

  * [User class](../classes/user.md)
    * [beginAuthSession()](../classes/user.md#beginauthsession)
    * [cancelAuthTicket()](../classes/user.md#cancelauthticket)
    * [endAuthSession()](../classes/user.md#endauthsession)
    * [getAuthSessionTicket()](../classes/user.md#getauthsessionticket)

---

## Setting Up Signals

First, in both your clients and server, you'll want to set two variables: **TICKET** and **CLIENT_TICKETS**. You will keep the local client's ticket dictionary in **TICKET**, obviously, and a list of all connected clients in your **CLIENT_TICKETS** array. More on that later.

````
# Set up some variables
var TICKET: Dictionary		# Your auth ticket
var CLIENT_TICKETS: Array	# Array of tickets from other clients
````

Now, we'll set up the signals for authentication callbacks:

=== "Godot 2.x, 3.x"
	````
	func _ready() -> void:
		Steam.connect("get_auth_session_ticket_response", self, "_get_Auth_Session_Ticket_Response")
		Steam.connect("validate_auth_ticket_response", self, "_validate_Auth_Ticket_Response")
	````
=== "Godot 4.x"
	````
	func _ready() -> void:
		Steam.get_auth_session_ticket_response.connect(_get_Auth_Session_Ticket_Response)
		Steam.validate_auth_ticket_response.connect(_validate_Auth_Ticket_Response)
	````

Next we implement the respective functions for when we receive the signals:

````
# Callback from getting the auth ticket from Steam
func _get_Auth_Session_Ticket_Response(auth_ticket: int, result: int) -> void:
	print("Auth session result: " + str(result))
	print("Auth session ticket handle: " + str(auth_ticket))
````

Our **_get_Auth_Session_Ticket_Response()** function will print out the auth ticket's handle and whether getting the ticket was successful (returns a `1`) or not. You can add logic for success or failure based on your game's needs. If successful, you'll probably want to send your new ticket to the server or another client for validation at this point.

Speaking of validation:

````
# Callback from attempting to validate the auth ticket
func _validate_Auth_Ticket_Response(authID: int, response: int, ownerID: int) -> void:
	print("Ticket Owner: " + str(authID))

	# Make the response more verbose, highly unnecessary but good for this example
	var VERBOSE_RESPONSE: String
	match response:
		0: VERBOSE_RESPONSE = "Steam has verified the user is online, the ticket is valid and ticket has not been reused."
		1: VERBOSE_RESPONSE = "The user in question is not connected to Steam."
		2: VERBOSE_RESPONSE = "The user doesn't have a license for this App ID or the ticket has expired."
		3: VERBOSE_RESPONSE = "The user is VAC banned for this game."
		4: VERBOSE_RESPONSE = "The user account has logged in elsewhere and the session containing the game instance has been disconnected."
		5: VERBOSE_RESPONSE = "VAC has been unable to perform anti-cheat checks on this user."
		6: VERBOSE_RESPONSE = "The ticket has been canceled by the issuer."
		7: VERBOSE_RESPONSE = "This ticket has already been used, it is not valid."
		8: VERBOSE_RESPONSE = "This ticket is not from a user instance currently connected to steam."
		9: VERBOSE_RESPONSE = "The user is banned for this game. The ban came via the Web API and not VAC."
	print("Auth response: " + str(VERBOSE_RESPONSE))
	print("Game owner ID: " + str(ownerID))
````

The **_validate_Auth_Ticket_Response()** function is received in response to **[beginAuthSession()](../classes/user.md#beginauthsession)** when the ticket has been validated. It sends back the Steam ID of the user being authorized, the result of the validation (success is `0` as shown above), and finally the Steam ID of the user that owns the game.

[As Valve notes](https://partner.steamgames.com/doc/api/ISteamUser#ValidateAuthTicketResponse_t){ target="_blank" }, the owner of the game and the user being authorized may be different if the game is borrowed from Steam Family Library Sharing. Inside this function, you can again put in whatever logic your game requires. You will probably want to add the client to the server if successful, obviously.

---

## Getting Your Auth Ticket

First you'll want to get an auth ticket from Steam and store it in your **TICKET** dictionary variable; this way you can pass it along to the server or other clients as needed:

````
TICKET = Steam.getAuthSessionTicket()
````

This function also has an optional **identity** you can pass to it but defaults to **null**. This identity can correspond to one of your networking identities set up with the [Networking Types](../classes/networking_types.md) class.

Now that you have your auth ticket, you'll want to pass it along to the server or other clients for validation.

---

## Validating the Auth Ticket

Your server or other clients will now want to take your auth ticket and validate it before allowing you to join the game. In a peer-to-peer situation, every client will want to validate the ticket of every other player. The server or clients will want to pass your **TICKET** dictionary's buffer and size, as well as your Steam ID, to **[beginAuthSession()](../classes/user.md#beginauthsession)**. For this we'll create a **_validate_Auth_Session()** function:

````
func _validate_Auth_Session(ticket: Dictionary, steam_id: int) -> void:
	var RESPONSE: int = Steam.beginAuthSession(ticket.buffer, ticket.size, steam_id)

	# Get a verbose response; unnecessary but useful in this example
	var VERBOSE_RESPONSE: String
	match RESPONSE:
		0: VERBOSE_RESPONSE = "Ticket is valid for this game and this Steam ID."
		1: VERBOSE_RESPONSE = "The ticket is invalid."
		2: VERBOSE_RESPONSE = "A ticket has already been submitted for this Steam ID."
		3: VERBOSE_RESPONSE = "Ticket is from an incompatible interface version."
		4: VERBOSE_RESPONSE = "Ticket is not for this game."
		5: VERBOSE_RESPONSE = "Ticket has expired."
	print("Auth verifcation response: "+str(VERBOSE_RESPONSE))

	if RESPONSE == 0:
		print("Validation successful, adding user to CLIENT_TICKETS")
		CLIENT_TICKETS.append({"id": steam_id, "ticket": ticket.id})
	
	# You can now add the client to the game

````

If the response is `0` (meaning the ticket is valid), you can allow the player to connect to the server or game. A callback will also be received and trigger our **_validate_Auth_Ticket_Response()** function which, as we saw before, sends along the Steam ID of the auth ticket provider, the result, and the Steam ID of the owner of the game. This callback will also be triggered when another user cancels their auth ticket. More on that later.

After the ticket is validated, you'll want to save the player's Steam ID and their ticket handle in your **CLIENT_TICKETS** array either as an array or dictionary so they can be called later to cancel the auth sessions. In our example above, we used a dictionary so we can just pull the ticket handle by the user's Steam ID.

---

## Canceling Auth Tickets and Ending Auth Sessions

Finally when the game is over or the client is leaving the game, you'll want to cancel your own auth ticket and end the auth sessions with other players.

When the client is ready to leave the game, they will pass their own ticket handle to the **[cancelAuthTicket()](../classes/user.md#cancelauthticket)** function like so:

````
Steam.cancelAuthTicket(TICKET.id)
````

This will trigger the **_validate_Auth_Ticket_Response()** function for the server or other clients to let them know the player has left and invalidated their auth ticket. Additionally, you should call **[endAuthSession()](../classes/user.md#endauthsession)** to also close out the auth session with the server or other clients:

````
Steam.endAuthSession(steam_id)
````

You will need to pass the Steam ID of every client connected. You can do this in a loop from your **CLIENT_TICKETS** array like so:

````
for CLIENT in CLIENT_TICKETS:
	Steam.endAuthSession(CLIENT.id)

	# Then remove this client from the ticket array
	CLIENT_TICKETS.erase(CLIENT)
````

The [Steamworks documentation](https://partner.steamgames.com/doc/features/auth){ target="_blank" } states that each player must cancel their own auth ticket(s) and end any auth sessions started with other players.

---

That concludes this simple tutorial for authenticated sessions.

To see this tutorial in action, [check out our GodotSteam Example Project on GitHub](https://github.com/CoaguCo-Industries/GodotSteam-Example-Project){ target="_blank" }. There you can get a full view of the code used which can serve as a starting point for you to branch out from.
