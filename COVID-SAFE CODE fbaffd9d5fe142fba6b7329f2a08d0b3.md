# COVID-SAFE CODE

We can divide the connection between Prolog and the user in the following: First, the user makes request to our Web Application using https. Then we load the HTML content, which starts a web socket and connects to our Python backend. In the backend, we run Prolog, load our KB, and start our query. For each askable, we send and wait for a response from our user. We unify each variable with the user response and repeat until we resolve the query. We send our result back to the user.

We ran the code by connecting Prolog to pyswip. To run pyswip using Django's Web Sockets, we had to change the threading of pyswip by inheriting its Prolog class and creating a new thread. This new thread allowed the front-end to communicate with the backend while still running Prolog unification.

Although the original intent was to deploy the code to production, we had trouble connecting Daphne to the Ubuntu Server (the production server). Daphne would permit use to use the WebSockets as we had done when we ran it locally.

We included the relevant parts of our code in the appendix. As it stands, the code cannot be rerun because it is implemented as part of a larger web app. Thus, we included only the work we did that is relevant and done for this particular LBA.

## Our Code

### Prolog KB

```prolog
%  Tell prolog that known/3 and multivalued/1 will be added later
:- dynamic known/3, multivalued/1.

% Enter your KB below this line:

multivalued(covid).
multivalued(outfit).

categoricalChoice(price).
categoricalChoice(mood).
categoricalChoice(energy).

club(sleep) :-
    ask_user(instruct,_,Y),
    Y \== yes.
    
club(stattbad) :- 
    (covid(vaccinated); covid(recovered)),
    \+population(high_risk),
    \+abroad(past_month),
    mask(ffptwo),
    \+residence(close),
    (price(twelve);price(fifteen)),
    (energy(mid); mood(chill)).

club(katerholzig) :- (covid(vaccinated); covid(recovered)),
    \+population(high_risk),
    \+abroad(past_month),
    mask(ffptwo),
    residence(close),
    price(fifteen),
    outfit(colorful),
    speaker(german),
    (energy(mid); energy(low)).

club(sisyphos) :- (covid(vaccinated); covid(recovered)),
    \+population(high_risk),
    \+abroad(past_month),
    mask(ffptwo),
    \+residence(close),
    (price(ten);price(twelve);price(fifteen)),
    \+group(big),
    (mood(chill); mood(happy); energy(low)).

club(salon_zur_wilden_renate) :- (covid(vaccinated); covid(recovered)),
    \+population(high_risk),
    \+abroad(past_month),
    mask(ffptwo),
    \+residence(close),
    (price(ten);price(twelve);price(fifteen)),
    speaker(german),
    \+group(big),
    mood(wild).

club(kitkat) :- (covid(vaccinated); covid(recovered)),
    \+population(high_risk),
    \+abroad(past_month),
    mask(ffptwo),
    residence(close),
    (price(ten);price(twelve);price(fifteen)),
    outfit(sexy),
    (mood(sexual); energy(high)). 

club(chalet) :- (covid(vaccinated); covid(recovered)),
    \+population(high_risk),
    \+abroad(past_month),
    mask(ffptwo),
    residence(close),
    (price(ten);price(twelve);price(fifteen)),
    speaker(german),
    (mood(chill); energy(mid)).

club(about_blank) :- (covid(vaccinated); covid(recovered)),
    \+population(high_risk),
    \+abroad(past_month),
    mask(ffptwo),
    \+residence(close),
    (price(ten);price(twelve);price(fifteen)),
    \+group(big),
    \+volume(loud),
    \+outfit(fancy),
    (mood(chill); energy(high)).

club(stattbad) :- (covid(vaccinated); covid(recovered)),
    \+population(high_risk),
    \+abroad(past_month),
    mask(ffptwo),
    \+residence(close),
    (price(ten);price(twelve);price(fifteen)),
    (energy(mid); mood(chill)).

% The code below implements the prompting to ask the user:

covid(X) :- ask(covid, X). % multivalued
population(X) :- ask(population, X).
abroad(X) :- ask(abroad, X).
mask(X) :- ask(mask, X).
residence(X) :- ask(residence, X).
outfit(X) :- ask(outfit, X). % multivalued
speaker(X) :- ask(speaker,X).
volume(X) :- ask(volume, X).
group(X) :- ask(group, X).
price(X) :- ask(price, X).
mood(X) :- ask(mood, X).
energy(X) :- ask(energy, X).

% Asking clauses

ask(A, V):-
    known(yes, A, V), % succeed if true
    !.      % stop looking

ask(A, V):-
    known(_, A, V), % fail if false
    !, fail.

% If not multivalued, and already known to be something else, dont ask again for a different value.

ask(A, V):-
    \+multivalued(A),
    known(yes, A, V2), % fail if has another value
    V \== V2, % which is not the same value
    !, fail.

ask(A, V):-
    categoricalChoice(A),  % if multiple choice
    ask_user(A,V,Y), % get an answer for Y
    assertz(known(yes, A, Y)), % send the answer as an atom
    V == Y. % succeed or fail

ask(A, V):-
    \+categoricalChoice(A),
    ask_user(A,V,Y), % get the answer
    assertz(known(Y, A, V)), % remember it
    Y == yes.       % succeed or fail
```

### PySwip

```python
#import necessary libraries
import json
import pyswip as psw
import threading
from pyswip.core import _findSwipl
from pyswip.core import CDLL
from ctypes import *

# Importing the libraries that pyswip uses to connect to Prolof
(_path, SWI_HOME_DIR) = _findSwipl()
nativeProlog = CDLL(_path, mode=RTLD_GLOBAL)

class Prolog(psw.Prolog):
		''' Inherit class from pyswip to change its threading and run it with web sockets
		'''
    def _init_prolog_thread(cls):
        x = threading.Thread(target=nativeProlog.PL_thread_self, args=[1], daemon=True)
        x.start()

def humanize_askable(A,V):
				''' Convert our askables to natural language
						We import the predicate and the atom
						We output question and its options (tuple)
				'''
        str_A, str_V = str(A), str(V)
        if str_A == "instruct":
            question = "Want to find a night club? Swipe right for yes. Swipe left for no."
            options = ["yes", "no"]
        elif str_A == "covid" and str_V == "vaccinated":
            question = "Are you vaccinated?"
            options = ["yes", "no"]
        elif str_A == "covid" and str_V == "recovered":
            question = "Have you recently recovered from covid?"
            options = ["yes", "no"]
        elif str_A == "population" and str_V == "high_risk":
            question = "Are covid levels at high risk where you are?"
            options = ["yes", "no"]
        elif str_A == "abroad" and str_V == "past_month":
            question = "Have you been abroad in the past month?"
            options = ["yes", "no"]
        elif str_A == "mask" and str_V == "ffptwo":
            question = "Are you willing to wear a mask?"
            options = ["yes", "no"]
        elif str_A == "residence" and str_V == "close":
            question = "Do you want somewhere close to the res?"
            options = ["yes", "no"]
        elif str_A == "outfit":
            question = f"Is your outfit {str_V}?"
            options = ["yes", "no"]
        elif str_A == "speaker" and str_V == "german":
            question = "Do you speak german?"
            options = ["yes", "no"]
        elif str_A == "volume" and str_V == "loud":
            question = "Are you ok with loud music?"
            options = ["yes", "no"]
        elif str_A == "group" and str_V == "big":
            question = "Are you going with a big group?"
            options = ["yes", "no"]
        elif str_A == "price":
            question = f"What is the most you want to spend in euros?"
            options = ["six","ten","twelve","fifteen"]
        elif str_A == "energy":
            question = f"How energetic are you feeling?"
            options = ["low","mid","high"] 
        elif str_A == "mood":
            question = f"What is your mood?"
            options = ["chill","happy","wild","sexual"] 
        else:
            question = f"{str_A}{str_V}"
            options = ["chill","happy","wild","sexual"] 
        
        return (question, options)

def initialiseProlog(w):
		''' Main running of Prolog,
				We use w to read and send messages to the user
				W is the other end of our miultithreading pipe
		'''

    prolog = Prolog() #initialize our Prolog class
    prolog._init_prolog_thread() #call our method
    
    def ask_user(A,V,Y): 
				''' Python function called by prolog.
						Given A,V, unify Y.
				'''
        if isinstance(Y, psw.Variable):
						#get askables in natural language
            question, options = humanize_askable(A,V)
						#send the user a dictionary with given information
            w.send({
                    'type': "question",
                    'question': question,
                    'options': options
                })
            readers = [w]
            while readers: #wait for the message from the websocket process running in parallel
                for r in wait(readers):
                    try:
                        msg = r.recv() #receive the message
                    except EOFError:
                        readers.remove(r)
                    else:
                        Y.unify(msg) #unify the user response with our askable
                        return True
        return False #return False if the user doesn't respond or Y is not a variable

    def resolve(club):
				''' Inputs the result of our query.
						Sends a message to the user about their match (dictionary)
						Closes the thread execution.
				'''
        if club:
            match = "You should go to " + club[0]['X'].capitalize() + "."
        else:
            match = "Go back to the res!"
        w.send({ #send message to the user that they have gotten a match
            'type': "match",
            'match': match,
        })
        w.close()

		#register our Python function to be called in our Prolog kb
    psw.registerForeign(ask_user, arity=3)

		#load our Prolog KB from our web app's static folder
    prolog.consult("static/prolog/clubKB.pl")

    club = [s for s in prolog.query("club(X).", maxresult=1)]

    resolve(club)
```

```python
#resource on websockets: https://channels.readthedocs.io/en/latest/topics/consumers.html
#resource on multithread: https://www.youtube.com/watch?v=fKl2JW_qrso
#resource on multithread: https://docs.python.org/3/library/multiprocessing.html

from channels.generic.websocket import WebsocketConsumer
from multiprocessing import Pipe, Process
from multiprocessing.connection import wait

class PrologConsumer(WebsocketConsumer):

		'''
			Websocket Consumer: What gets called when we initialize the connection
			A class where we can send and receive messages without having to reload the page.
			Not through http but ws. 
		'''

    def __init__(self, *args, **kwargs):

        r, w = Pipe(duplex=True) #create a pipe where both sides can send and receive messages
        self.readers = [] # a queue so prolog knows what is running and whether it should wait for a message
        self.r = r
        self.readers.append(self.r)
				#the thread, where target is the function and arg the arguments we will pass when hit .start()
        self.p = Process(target=initialiseProlog, args=[w]) 
        self.p.start() #start the function
        w.close() #close the w end of the pipe

    def connect(self):
				''' on connection with our user (when page has loaded)
				'''
        self.accept()
        self.await_message()
        
    def await_message(self):
				''' Here we wait for Prolog to send queries
				'''
        for r in wait(self.readers):
            try:
                msg = r.recv() #Prolog query saved as msg
            except EOFError:
                self.readers.remove(r)
            else:
                self.send(json.dumps(msg)) # send message in json format to React Js

    
    def disconnect(self, close_code):
				''' If user gets disconnected or closes the page, terminate thread (automatically closes all connections)
				'''
        self.p.terminate()

    def receive(self, user_response):
				''' Receive data from user
				'''
        text_data_json = json.loads(user_response)
        message = text_data_json['message'] #only save the message from our text
        self.r.send(message) #send data to Prolog
        self.await_message() #await Prolog query
```

### Relevant Django Connections

```python
# urls.py, how we direct the user given the typed url

from django.urls import path
from . import views

urlpatterns = [
    path('cs152',views.prologLBA) #send to views.prologLBA
]

# views.py 

def prologLBA(request):
	''' From user request, load the html page
	'''
    return render(request,"cs113/prologLBA.html")

#asgi.py, where we make the asgi connections

import os
from channels.auth import AuthMiddlewareStack
from channels.security.websocket import AllowedHostsOriginValidator
from channels.routing import ProtocolTypeRouter, URLRouter
from django.core.asgi import get_asgi_application
import core.routing

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'pablo.settings')

application = ProtocolTypeRouter({
    "http": get_asgi_application(),
    "websocket": AllowedHostsOriginValidator(
        AuthMiddlewareStack(
            URLRouter(
                core.routing.websocket_urlpatterns
            )
        ),
    ),
})

# routing.py

from django.urls import re_path
from cs113.consumers import ChatConsumer

websocket_urlpatterns = [
    re_path(r'ws/chat', ChatConsumer.as_asgi()),
]
```

3. User Interface

```html
{% load static %} <!-- Load relevant libraries for HTML with django -->
{% load compress %}

<!DOCTYPE html>
<html lang="en">

    <head>
	      <!-- Ensure content is not scrollable  -->
        <meta content="width=device-width, user-scalable=no" name="viewport" />
        <!-- Link to stylesheet (formatting colors, animations, etc  -->
				<link rel="stylesheet" href="{% static 'cs113/css/cs152.css' %}">      
    </head>

    <body>

				<!-- Empty divs which we will fill with JS  -->
        <div id="fakeBody">
        </div>

        <div id="optionChosen">
        </div>

				<!-- Load JS using compress so we can use modules like react -->
        {% compress js %}
        <script type="module" src="{% static "cs113/js/prologConnection.js" %}"></script>
        {% endcompress %}

    </body>

</html>
```

```css
body, textarea, button, form, input{ /* Custom Font */
    font-family:'Gill Sans', 'Gill Sans MT', Calibri, 'Trebuchet MS', sans-serif;
}

body { /* Main Body */
    background: linear-gradient(180deg, #0B0205 0%, #3F2D33 100%);
    width:100vw;
    height:100vh;
    overflow:hidden;
    padding:0px;
    margin:0px;
    overflow:-moz-hidden-unscrollable;
}

#fakeBody{ /* Body to load the React Components */
    width:100vw;
    height:90vh;
    display:flex;
    justify-content: center;
    padding:0px;
    box-sizing: border-box;
    overflow:hidden;
    overflow:-moz-hidden-unscrollable;
}

.cardContainer{ 
    box-sizing: border-box;
    width:100vw;
    position:fixed;
    top:0px;
    padding:0px;
    display:flex;
    justify-content:center;
    overflow:hidden;
    overflow:-moz-hidden-unscrollable;
}

/* Animations everytime our cards load */
@keyframes appear{
    0%{
        opacity:0;
        transform:scale(0.9);
    }
    50%{
        opacity:0;
        transform:scale(0.9);
    }
    100%{
        opacity:1;
        transform:scale(1);
    }
}

.card.appear{
    animation: appear 1s linear 1; /* animation name, length, linear interpolation, how many times we run it*/
}

.card{ /* Card for each askable */
    width:80vw;
    height:70vh;
    background: linear-gradient(180deg, #AA076B 0%, #61045F 100%);
    border-radius:20px;
    padding:20px;
    margin-top:5vh;
    margin-left:auto;
    margin-right:auto;
    display:flex;
    align-items:center;
    flex-direction: column;
    justify-content: space-around;
    overflow:hidden;
}

.card.match{ /* Customize card if the card also has the class match */
    justify-content:first baseline;
    animation: appear 0.4s linear 1;
    background: linear-gradient(180deg, #aaffa9 0%, #11ffbd 100%); /* Green background */
}

.card.match .match_label{
    font-size:24px;
    justify-self:flex-start;
    color:white;
}

@keyframes disappearR{ /* When user scrolls right */
    0%{
        transform: scale(1) translate(initial, 0) rotate(0rad);
    }
    100%{
        opacity:0;
        transform: scale(0.9) translate(100vw, 0) rotate(1.7rad);
    }
}

@keyframes disappearL{ /* When user scrolls left */
    0%{
        transform: scale(1) translate(initial, 0) rotate(0rad);
    }
    100%{
        opacity:0;
        transform: scale(0.9) translate(-100vw, 0) rotate(-1.7rad);
    }
}

.card.disappearR{ /* When user scrolls right */
    animation : none;
    animation: disappearR 2s cubic-bezier(0.075, 0.82, 0.165, 1) forwards;
}

.card.disappearL{ /* When user scrolls right */
    animation : none;
    animation: disappearL 2s cubic-bezier(0.075, 0.82, 0.165, 1) forwards;
}

.askable{ /* Askable text */
    color:white;
    text-align: center;
    font-size:18px;
}

#optionChosen{ /* Options when multiple choice */
    position:fixed;
    color:aliceblue;
    opacity:0.8;
    bottom:10vh;
    z-index:10;
    font-size:30px;
    text-align: center;
    width:100vw;
}

.multi_choice{ /* Container for each option */
    display:flex;
    flex-direction: row;
    width:100%;
    justify-content: space-evenly;
}

.multi_choice button{ /* Customize buttons for each option */
    color:white;
    font-size:14px;
    background:none;
    border:1px solid white;
    border-radius:20px;
    padding:4px 14px;
}
```

```jsx
import React from "react";
import ReactDOM from "react-dom";
import Draggable from "react-draggable";
import $ from 'jquery';

function buildID() {
		/* Build a new ID for each div element.
			Source from https://stackoverflow.com/questions/1349404/generate-random-string-characters-in-javascript
		*/
    var ID = "";
    var choices = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
    for (let i = 0; i < 5; i++) //given an ID of length 5, pick a random value from choices str
        ID += choices.charAt(Math.floor(Math.random() * choices.length));
    return ID;
  }

function makeWS(){
		/* How we connect to our websocket.
		*/

		// first ensure ws or wss given how our user connects to our web app (https vs http)
    const protocol = (window.location.protocol == 'https:') ? 'wss://' : 'ws://';
    const chatSocket = new WebSocket(
        protocol + window.location.host + '/ws/chat'
    ); // instantiate class given the connection for Prolog
    
    chatSocket.onmessage = function(e) { // what to do when we receive a message
        
				const data = JSON.parse(e.data); // load JSON data
        const newID = buildID(); // make a new ID for our DIV
        const newDiv = document.createElement("div"); // create DIV
        newDiv.setAttribute('class','cardContainer'); // add cardContainer class
        document.getElementById("fakeBody").appendChild(newDiv);

				// given question, or match load new React component
				// for each component pass in given attributes received from back-end
        const renderin = (data.type == "question") ?
            <Card  divID={newID} ws={chatSocket}
            askable={data.question}
            options={data.options} />
            :
            <Match  divID={newID} ws={chatSocket}
            match={data.match} />

        ReactDOM.render(renderin, newDiv); // render the DIV
    };
    
    chatSocket.onclose = function(e) { // if socket closes on backend
        console.error('Chat socket closed unexpectedly');
    };
    
}

$(document).ready(function(){ // when the HTML finishes loading, establish ws connection

    makeWS(); // WS = Web Socket

});

    
class Card extends React.Component {

		// Create React Component Class for each new askable

    constructor(props) {
        super(props); // get extra prop attributes that won't change 
        this.ws = props.ws; // web socket reference
        this.state = { // states that changes 
            hasChosen: 0, // whether user chose option
            binaryChoice: this.props.options.length==2 // whether option is multiple choice
        }
				// using bind so we can refer to the instance and its props using "this" inside each function
        this.handleDrag = this.handleDrag.bind(this); 
        this.handleDragSubmit = this.handleDragSubmit.bind(this);
        this.handleClickSubmit = this.handleClickSubmit.bind(this);
    };

    handleDrag(){
				// when user drags a card
        if(this.state.hasChosen == 0){ // stop dragging if choice selected
						
						// get card position
            const cardX = $("#".concat(this.props.divID)).css('transform').split(',')[4];

						// normalize position
            const cardRelativeWidth = 0.9;
            const cardRelativeX = cardX/window.screen.width*cardRelativeWidth;
            
						$("#".concat(this.props.divID)).css('opacity',(1-Math.abs(cardRelativeX)));
            
						// render option chosen given is position on left (negative value) or right (positive value)
						const optionChosen = (cardRelativeX > 0) ? "yes" : "no";
            $("#optionChosen").html(`option chosen: ${optionChosen}`);
					
						// send message if user confidence over certain threshold
            if (Math.abs(cardRelativeX) > 0.35){
                this.handleDragSubmit(optionChosen); // yes or no
                if (cardRelativeX > 0) { // choice given left (-) or right hand side (+)
                    this.setState({ hasChosen: 1 });
                } else {
                    this.setState({ hasChosen: -1 });
                }
            }
        }
        
    };
    
    handleDragSubmit(output){
				// send to websocket as json format 
        this.ws.send(JSON.stringify({
            'message': output
        }));
    };

    handleClickSubmit(e){
				/* when user clicks on option,
							change its color and send value of button to unify Prolog value
				*/

        $(e.target).css("background","white"); // change button color
        $(e.target).css("color","#61045F"); // change button text color

        $("#optionChosen").html(`option chosen: ${e.target.value}`);
        setTimeout(()=>{ // after 600 ms send the message and change the card state
            this.ws.send(JSON.stringify({
                'message': e.target.value
            }));
            this.setState({ hasChosen: 1 });
        },600) // time out to allow the user 600 ms to see their option change color
        
    };
    
    render() { // render component visible
			
        var draggableClass = "card"; 
        switch(this.state.hasChosen){ // set class depending on chosen attribute
            case 0: // not chosen
                draggableClass = "card appear"
                break
            case 1: // has chosen yes, so card flicks right
                draggableClass = "card disappearR"
                break
            case -1: // has chosen yes, so card flicks left
                draggableClass = "card disappearL"
                break
        }

        return (
        <Draggable // draggable component
            axis="x" // only drag on x axis, no y axis movement
            onDrag={this.handleDrag} // function to call on drag
						// disable drag if multiple choice or user already chose an option
            disabled={(this.state.hasChosen != 0)||(!this.state.binaryChoice)} 
         >
            <div id={this.props.divID}
                className={draggableClass}>
                <div className="askable">
                    {this.props.askable} {/* question user gets asked */}
                </div>
                { (!this.state.binaryChoice) && // render multiple choice buttons
                    <div className="multi_choice">
                        {this.props.options.map((option,key) =>  // for each choice, create a button
                        (<div key={key}>
                            <button value={option} onClick={this.handleClickSubmit}>
                                {option}
                            </button>
                         </div>))}
                    </div>
                }
            </div>
        </Draggable>
  );}
}

class Match extends React.Component {

		// React Component To Load the match we get from our query

    constructor(props) {
        super(props);
    };
    
    render() {
        return (
        <div>
            <div id={this.props.divID} className="card match">
                <div className="match_label">You have a match!</div>
                <div className="askable">
                    {this.props.match} {/* Display match message */}
                </div>
            </div>
        </div>
  );}
}
```