# number-guessing-game-
#!/bin/bash

# PostgreSQL connection string
PSQL="psql --username=freecodecamp --dbname=number_guess -t --no-align -c"

# Function to check if username exists
check_user_exists() {
  local username=$1
  user_exists=$($PSQL "SELECT COUNT(*) FROM users WHERE username = '$username';")
  if [[ $user_exists -eq 0 ]]
  then
    return 1  # User doesn't exist
  else
    return 0  # User exists
  fi
}

# Function to get user data (games_played, best_game)
get_user_data() {
  local username=$1
  user_data=$($PSQL "SELECT games_played, best_game FROM users WHERE username = '$username';")
  games_played=$(echo $user_data | cut -d'|' -f1)
  best_game=$(echo $user_data | cut -d'|' -f2)
}

# Function to insert new user into the database
insert_new_user() {
  local username=$1
  $($PSQL "INSERT INTO users (username) VALUES ('$username');")
  echo "Welcome, $username! It looks like this is your first time here."
}

# Function to update the best game (fewest guesses)
update_best_game() {
  local username=$1
  local guesses=$2
  current_best_game=$($PSQL "SELECT best_game FROM users WHERE username = '$username';")
  
  if [[ -z $current_best_game || $guesses -lt $current_best_game ]]
  then
    $($PSQL "UPDATE users SET best_game = $guesses WHERE username = '$username';")
  fi
}

# Prompt for the username
echo "Enter your username:"
read username

# Check if the user exists
check_user_exists $username
if [[ $? -eq 0 ]]
then
  # User exists, get their data
  get_user_data $username
  echo "Welcome back, $username! You have played $games_played games, and your best game took $best_game guesses."
else
  # User doesn't exist, insert a new user
  insert_new_user $username
fi

# Generate the secret number (between 1 and 1000)
secret_number=$((RANDOM % 1000 + 1))
guesses=0

# Ask the user to guess the secret number
echo "Guess the secret number between 1 and 1000:"

# Start the guessing game
while true; do
  read guess

  # Check if the guess is a valid integer
  if ! [[ "$guess" =~ ^[0-9]+$ ]]
  then
    echo "That is not an integer, guess again:"
  elif [[ $guess -lt $secret_number ]]
  then
    echo "It's higher than that, guess again:"
  elif [[ $guess -gt $secret_number ]]
  then
    echo "It's lower than that, guess again:"
  else
    # Correct guess, display the success message
    guesses=$((guesses + 1))
    echo "You guessed it in $guesses tries. The secret number was $secret_number. Nice job!"
    
    # Update the user's data in the database
    update_best_game $username $guesses
    $($PSQL "UPDATE users SET games_played = games_played + 1 WHERE username = '$username';")
    
    # Finish the game
    break
  fi

  # Increment guesses
  guesses=$((guesses + 1))
done
