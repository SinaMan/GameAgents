from api import commander
import random
import threading

from api import orders
from api.vector2 import Vector2


#Main Function Commander
class MyCommander(commander.Commander):
    """
    Leaves everyone to defend the flag except for one lone guy to grab the other team's flag.
    """
	
    def initialize(self):
        self.state = BalancedCommander()    # Initialize a Balanced Commander
        self.state.initialize(self)
        self.gameIsStuck = False

    #check when there is a draw
    def gameStuck(self):
        if self.game.match.scores.get('Blue') - self.game.match.scores.get('Red') == 0:
            self.gameIsStuck = True

    def tick(self):
        # after 90 seconds check if the game is "stuck"
        # threading.Timer(90.0, self.gameStuck).start()

        blueScore = self.game.match.scores.get('Blue')
        redScore = self.game.match.scores.get('Red')

        """Verify conditions to switch commander"""

        # When we are winning-> we go completely in defense mode
        if self.state.type != 'DEFENDER' and blueScore > redScore:
            self.gameIsStuck = False
            print("Blue Team Switching to Defender Commander")
            self.state = DefenderCommander()
            self.state.initialize(self)
        # When we are winning (2 points of difference) -> we can afford to attack again
		# self.state.type == 'DEFENDER' and
        if  self.state.type != 'BALANCED' and blueScore == redScore :
            self.gameIsStuck = False
            print("Switching to Balanced Commander")
            self.state = BalancedCommander()
            self.state.initialize(self)

        # When we are loosing of 1 point -> we have to attack
        if self.state.type != 'ATTACKER' and redScore - blueScore == 1:
            self.gameIsStuck = False
            print("Switching to Attacker Commander")
            self.state = AttackerCommander()
            self.state.initialize(self)
        # When we are loosing of 2 or more points OR the gameIsStuck -> everyone attack
        if self.state.type != 'GREEDY' and (redScore - blueScore >= 2 or self.gameIsStuck):
            print("Switching to Greedy Commander")
            self.gameIsStuck = False
            self.state = GreedyCommander()
            self.state.initialize(self)

        # execute the behavior
        self.state.tick()

class AttackerCommander():

    def initialize(self, commander):
        self.commander = commander
        #Specify the state we are in
        self.type = 'ATTACKER'
        self.defender = None


    def captured(self):
        """Did this team capture the enemy flag?"""
        return self.commander.game.enemyTeam.flag.carrier != None

    def tick(self):
        """Two ways of attacking pattern, random choice between charge and slow attack. One bot will always be defend"""

        #Simplified commands
        captured = self.captured()
        commander = self.commander
        targetLocation = commander.game.team.flagScoreLocation
        enemyFlagLocation = commander.game.enemyTeam.flag.position
        our_flag = commander.game.team.flag.position
		
		#when we have no defender or dead, we need one when free
        if self.defender and (self.defender.health <= 0):
            self.defender = None

        # First process bots that are done with their orders or they don't have any order yet
        for bot in commander.game.bots_available:
            if self.defender == None and self.commander.game.enemyTeam.flag.carrier != bot:
                self.defender = bot

                targetMin = our_flag - Vector2(2.0, 2.0)
                targetMax = our_flag + Vector2(2.0, 2.0)
                position = commander.level.findRandomFreePositionInBox((targetMin,targetMax))
                if (our_flag-bot.position).length() > 2:
                    commander.issue(orders.Charge, self.defender, position, description = 'run to the flag')
                else:
                    commander.issue(orders.Attack, bot, position, description = 'defend around flag')
            else:
                if self.defender == bot:
                        self.defender = None
                if captured:
                    # Return the flag home
                    targetMin = targetLocation - Vector2(4.0, 4.0)
                    targetMax = targetLocation + Vector2(4.0, 4.0)
                    position = commander.level.findRandomFreePositionInBox((targetMin,targetMax))
                    commander.issue(orders.Charge, bot, position, description = 'return enemy flag!')
                else:
                    # Find the enemy team's flag position and run to that.
                    if random.choice([True, False]):
                        commander.issue(orders.Attack, bot, enemyFlagLocation, description = 'Attack flag!')
                    else:
                        commander.issue(orders.Charge, bot, enemyFlagLocation, description = 'Charge flag!')

        for bot in commander.game.bots_holding:
            if captured:
                targetLocation = commander.game.team.flagScoreLocation
                commander.issue(orders.Charge, bot, targetLocation , description = 'return enemy flag!')
            else:
                commander.issue(orders.Charge, bot, enemyFlagLocation, description = 'Charge flag!')


class BalancedCommander():

    def initialize(self, commander):
        self.commander = commander
        self.type = 'BALANCED'
        self.defender = None


    def tick(self):
        """Two teams, either send them to attack or to defend. Place a little emphasis on defense"""

        commander = self.commander
        our_flag = commander.game.team.flag.position
        targetLocation = commander.game.team.flagScoreLocation

        if self.defender and (self.defender.health <= 0 or self.defender.flag):
            self.defender = None

        # First process bots that are done with their orders...
        for bot in commander.game.bots_available:



            if ( self.defender == None or bot.flag) and len(commander.game.bots_alive) > 1:
                self.defender = bot
                targetMin = our_flag - Vector2(2.0, 2.0)
                targetMax = our_flag + Vector2(2.0, 2.0)
                position = commander.level.findRandomFreePositionInBox((targetMin,targetMax))
                if position:
                    their_flag = commander.game.enemyTeam.flag.position
                    their_base = commander.level.botSpawnAreas[commander.game.enemyTeam.name][0]
                    their_score = commander.game.enemyTeam.flagScoreLocation
                    commander.issue(orders.Defend, self.defender, [(p-bot.position, t) for p, t in [(our_flag, 5.0), (their_flag, 2.5), (their_base, 2.5), (their_score, 2.5)]], description = 'defending by scanning')

            # If we captured the flag
            if commander.game.enemyTeam.flag.carrier != None:
                # Return the flag home relatively quickly!
                targetMin = targetLocation - Vector2(2.0, 2.0)
                targetMax = targetLocation + Vector2(2.0, 2.0)
                position = commander.level.findRandomFreePositionInBox((targetMin,targetMax))
                commander.issue(orders.Charge, bot, position, description = 'returning enemy flag!')
            # In this case, the flag has not been captured yet
            else:
                path = [commander.game.enemyTeam.flag.position]

                if random.choice([True, False]):
                    targetPosition = commander.game.team.flag.position
                    targetMin = targetPosition - Vector2(8.0, 8.0)
                    targetMax = targetPosition + Vector2(8.0, 8.0)
                    position = commander.level.findRandomFreePositionInBox((targetMin,targetMax))
                    if position and (targetPosition - position).length() > 3.0:
                        commander.issue(orders.Charge, bot, position, description = 'defending the flag')
                else:
                    commander.issue(orders.Charge, bot, path, description = 'attacking enemy flag')


        # Process the bots that are waiting for orders, bots are in a holding attack pattern.
        holding = len(commander.game.bots_holding)
        for bot in commander.game.bots_holding:
            if holding > 3:
                commander.issue(orders.Charge, bot, random.choice([b.position for b in bot.visibleEnemies]))


class GreedyCommander():
    """
    Charge all bots to the flag of the enemy, we want to score fast
    """

    def initialize(self, commander):
        self.commander = commander
        self.type = 'GREEDY'

    def captured(self):
        """Did this team cature the enemy flag?"""
        return self.commander.game.enemyTeam.flag.carrier != None

    def tick(self):
        commander = self.commander
        captured = self.captured()
        target = commander.game.team.flagScoreLocation

        for bot in commander.game.bots_available:

            # If this team has captured the flag, then tell this bot...
            if captured:
                if bot.flag :
                    commander.issue(orders.Move, bot, target, description='return home')
                else:
                    # run to the exact flag location and escort the carrier.
                    commander.issue(orders.Attack, bot, commander.game.enemyTeam.flag.position, description='defending flag')

            # In this case, the flag has not been captured yet so have this bot attack it!
            else:
                path = [commander.game.enemyTeam.flag.position]
                commander.issue(orders.Charge, bot, path, description='charging enemy flag')

        # Process the bots that are waiting for orders, bots are in a holding attack pattern.
        holding = len(commander.game.bots_holding)
        for bot in commander.game.bots_holding:
            #if there is only 1 holding stay there and kill
            if holding > 1:
                commander.issue(orders.Charge, bot, random.choice([b.position for b in bot.visibleEnemies]))
            else:
                #spread out
                target = commander.level.findRandomFreePositionInBox((bot.position-5.0, bot.position+5.0))
                if target:
                    commander.issue(orders.Attack, bot, target, lookAt = random.choice([b.position for b in bot.visibleEnemies]))

class DefenderCommander():
    """
    Leaves everyone to defend the flag except for one lone guy to grab the other team's flag.
    """

    def initialize(self, commander):
        self.type = 'DEFENDER'
        self.commander = commander
        self.attacker = None

    def tick(self):

        commander = self.commander
        targetLocation = commander.game.team.flagScoreLocation
        enemyFlagLocation = commander.game.enemyTeam.flag.position

        if self.attacker and self.attacker.health <= 0:
            self.attacker = None

        for bot in commander.game.bots_available:

            if (self.attacker == None or bot.flag) and len(commander.game.bots_alive) > 1:
                self.attacker = bot

                if bot.flag:
                    # Return the flag home
                    commander.issue(orders.Charge, bot, targetLocation, description = 'returning enemy flag!')

                else:
                    # Find the enemy team's flag position and run to that.
                    commander.issue(orders.Charge, bot, enemyFlagLocation, description = 'take enemy flag!')

            else:
                if self.attacker == bot:
                    self.attacker = None

                # defend the flag!
                targetPosition = commander.game.team.flag.position
                targetMin = targetPosition - Vector2(8.0, 8.0)
                targetMax = targetPosition + Vector2(8.0, 8.0)
                position = commander.level.findRandomFreePositionInBox((targetMin,targetMax))

                if (targetPosition - bot.position).length() > 9.0:
                    if position:
                        commander.issue(orders.Charge, bot, position, description = 'Protect our flag')
                else:
                    their_flag = commander.game.enemyTeam.flag.position
                    their_base = commander.level.botSpawnAreas[commander.game.enemyTeam.name][0]
                    their_score = commander.game.enemyTeam.flagScoreLocation
                    commander.issue(orders.Defend, bot, [(p-bot.position, t) for p, t in [(targetPosition, 5.0), (their_flag, 2.5), (their_base, 2.5), (their_score, 2.5)]], description = 'defending by scanning')
