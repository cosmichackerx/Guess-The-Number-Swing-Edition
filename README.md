# Guess-The-Number-Swing-Edition

```java
import javax.swing.*;
import javax.swing.border.EmptyBorder;
import java.awt.*;
import java.awt.event.*;
import java.io.*;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.Properties;
import java.util.Random;

/**
 * GuessTheNumberSwing.java
 *
 * A polished Swing GUI version of the "Guess the Number" game.
 * Features:
 *  - Difficulty selector (Easy/Medium/Hard)
 *  - Attempt counting and limits
 *  - Hints (higher/lower + hot/warm/cold)
 *  - High-score persistence (best attempts per difficulty)
 *  - Play again / New game menu and keyboard-friendly input
 *
 * To run: compile and run this file with Java 8+:
 *   javac GuessTheNumberSwing.java
 *   java GuessTheNumberSwing
 */
public class GuessTheNumberSwing extends JFrame {
    private static final String HS_FILE = "guess_highscores.properties";

    private final Random rnd = new Random();
    private int secret;
    private int max;
    private int attempts;
    private int maxGuesses;
    private boolean inGame = false;

    // UI components
    private final JLabel titleLabel = new JLabel("Guess The Number");
    private final JComboBox<String> difficultyCombo = new JComboBox<>(new String[]{"Easy (1-20)", "Medium (1-100)", "Hard (1-1000)"});
    private final JButton startButton = new JButton("Start Game");
    private final JLabel statusLabel = new JLabel("Press Start to begin", SwingConstants.CENTER);
    private final JLabel hintLabel = new JLabel("", SwingConstants.CENTER);
    private final JTextField guessField = new JTextField();
    private final JButton guessButton = new JButton("Guess");
    private final JLabel attemptsLabel = new JLabel("Attempts: 0 / 0", SwingConstants.CENTER);
    private final JButton giveUpButton = new JButton("Give Up");
    private final JLabel highScoreLabel = new JLabel("High Scores — Easy: N/A  Medium: N/A  Hard: N/A", SwingConstants.CENTER);

    private final Properties highscores = new Properties();

    public GuessTheNumberSwing() {
        super("Guess The Number — Swing Edition");
        loadHighScores();
        buildUI();
        attachListeners();
        pack();
        setLocationRelativeTo(null);
        setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        setResizable(false);
    }

    private void buildUI() {
        setLayout(new BorderLayout());

        // Top: Title
        titleLabel.setFont(new Font("SansSerif", Font.BOLD, 26));
        titleLabel.setHorizontalAlignment(SwingConstants.CENTER);
        titleLabel.setBorder(new EmptyBorder(12, 12, 12, 12));
        add(titleLabel, BorderLayout.NORTH);

        // Center: Status and hint
        JPanel center = new JPanel(new GridLayout(4, 1, 6, 6));
        center.setBorder(new EmptyBorder(10, 20, 10, 20));
        statusLabel.setFont(new Font("SansSerif", Font.PLAIN, 16));
        center.add(statusLabel);
        center.add(hintLabel);
        hintLabel.setFont(new Font("SansSerif", Font.PLAIN, 14));
        hintLabel.setForeground(new Color(60, 60, 60));

        // Attempt/Highscore row
        JPanel infoRow = new JPanel(new BorderLayout(10, 0));
        infoRow.add(attemptsLabel, BorderLayout.WEST);
        infoRow.add(highScoreLabel, BorderLayout.CENTER);
        center.add(infoRow);

        // Difficulty + start
        JPanel controls = new JPanel();
        controls.add(difficultyCombo);
        controls.add(startButton);
        center.add(controls);

        add(center, BorderLayout.CENTER);

        // Bottom: Guess input
        JPanel bottom = new JPanel(new BorderLayout(6, 6));
        bottom.setBorder(new EmptyBorder(10, 20, 14, 20));

        guessField.setFont(new Font("SansSerif", Font.PLAIN, 16));
        guessField.setColumns(8);
        JPanel left = new JPanel();
        left.add(new JLabel("Your guess:"));
        left.add(guessField);
        bottom.add(left, BorderLayout.WEST);

        JPanel right = new JPanel();
        right.add(guessButton);
        right.add(giveUpButton);
        bottom.add(right, BorderLayout.EAST);

        add(bottom, BorderLayout.SOUTH);

        // Menu
        JMenuBar menuBar = new JMenuBar();
        JMenu gameMenu = new JMenu("Game");
        JMenuItem newItem = new JMenuItem("New Game");
        JMenuItem resetHS = new JMenuItem("Reset High Scores");
        JMenuItem exitItem = new JMenuItem("Exit");
        gameMenu.add(newItem);
        gameMenu.add(resetHS);
        gameMenu.addSeparator();
        gameMenu.add(exitItem);
        menuBar.add(gameMenu);
        setJMenuBar(menuBar);

        // Default states
        setGameState(false);

        // Menu actions
        newItem.addActionListener(e -> startNewGame());
        resetHS.addActionListener(e -> {
            int ok = JOptionPane.showConfirmDialog(this, "Reset all high scores?", "Confirm", JOptionPane.YES_NO_OPTION);
            if (ok == JOptionPane.YES_OPTION) {
                highscores.clear();
                saveHighScores();
                updateHighScoreLabel();
            }
        });
        exitItem.addActionListener(e -> System.exit(0));

        updateHighScoreLabel();
    }

    private void attachListeners() {
        startButton.addActionListener(e -> startNewGame());
        guessButton.addActionListener(e -> submitGuess());
        giveUpButton.addActionListener(e -> giveUp());

        // Enter key in guessField
        guessField.addActionListener(e -> submitGuess());

        // Accessibility: Ctrl+N to new game
        getRootPane().getInputMap(JComponent.WHEN_IN_FOCUSED_WINDOW).put(KeyStroke.getKeyStroke(KeyEvent.VK_N, InputEvent.CTRL_DOWN_MASK), "newGame");
        getRootPane().getActionMap().put("newGame", new AbstractAction() {
            @Override
            public void actionPerformed(ActionEvent e) {
                startNewGame();
            }
        });
    }

    private void startNewGame() {
        inGame = true;
        attempts = 0;
        int diff = difficultyCombo.getSelectedIndex();
        switch (diff) {
            case 0 -> max = 20;
            case 2 -> max = 1000;
            default -> max = 100; // medium
        }
        secret = rnd.nextInt(max) + 1;
        maxGuesses = Math.max(5, (int) Math.ceil(Math.log(max) / Math.log(2)) + 1);
        setGameState(true);

        statusLabel.setText(String.format("I picked a number between 1 and %d. You have %d attempts.", max, maxGuesses));
        hintLabel.setText("Make your first guess — go on, impress me.");
        guessField.setText("");
        guessField.requestFocusInWindow();
        updateAttemptsLabel();

        // subtle visual cue: clear background
        getContentPane().setBackground(null);
    }

    private void setGameState(boolean playing) {
        inGame = playing;
        guessButton.setEnabled(playing);
        guessField.setEnabled(playing);
        giveUpButton.setEnabled(playing);
        startButton.setEnabled(!playing);
        difficultyCombo.setEnabled(!playing);
    }

    private void submitGuess() {
        if (!inGame) return;
        String text = guessField.getText().trim();
        if (text.isEmpty()) {
            flashLabel(hintLabel, "Type a number first!");
            return;
        }
        int guess;
        try {
            guess = Integer.parseInt(text);
        } catch (NumberFormatException ex) {
            flashLabel(hintLabel, "That's not a valid number, silly.");
            return;
        }

        attempts++;
        updateAttemptsLabel();

        if (guess < 1 || guess > max) {
            flashLabel(hintLabel, String.format("Guess must be between 1 and %d — that's not even in the game.", max));
            return;
        }

        if (guess == secret) {
            winSequence();
            return;
        }

        if (guess < secret) {
            hintLabel.setText("Too low. Aim higher! ↑");
        } else {
            hintLabel.setText("Too high. Aim lower! ↓");
        }

        // proximity hint
        int diff = Math.abs(secret - guess);
        String proximity;
        if (diff <= Math.max(1, max / 20)) proximity = "You're burning hot! 🔥";
        else if (diff <= Math.max(2, max / 10)) proximity = "Warm. ♨️";
        else proximity = "Cold. 🥶";
        hintLabel.setText(hintLabel.getText() + " " + proximity);

        if (attempts >= maxGuesses) {
            loseSequence();
        } else {
            guessField.selectAll();
            guessField.requestFocusInWindow();
        }
    }

    private void giveUp() {
        if (!inGame) return;
        int ok = JOptionPane.showConfirmDialog(this, "Give up and reveal the number?", "Give Up?", JOptionPane.YES_NO_OPTION);
        if (ok == JOptionPane.YES_OPTION) {
            JOptionPane.showMessageDialog(this, "The number was " + secret + ". Don't be sad, try Easy next time 😏");
            endGame(false);
        }
    }

    private void winSequence() {
        statusLabel.setText(String.format("🎉 Correct! The number was %d. Attempts: %d", secret, attempts));
        flashBackground(new Color(198, 255, 200));
        updateHighScoreIfNeeded();
        endGame(true);
    }

    private void loseSequence() {
        statusLabel.setText(String.format("💥 Out of attempts! The number was %d.", secret));
        flashBackground(new Color(255, 220, 220));
        endGame(false);
    }

    private void endGame(boolean won) {
        inGame = false;
        setGameState(false);
        if (won) {
            int replay = JOptionPane.showConfirmDialog(this, "You did it! Play again?", "Winner", JOptionPane.YES_NO_OPTION);
            if (replay == JOptionPane.YES_OPTION) startNewGame();
        } else {
            int replay = JOptionPane.showConfirmDialog(this, "Play again?", "Try again", JOptionPane.YES_NO_OPTION);
            if (replay == JOptionPane.YES_OPTION) startNewGame();
        }
    }

    private void updateAttemptsLabel() {
        attemptsLabel.setText(String.format("Attempts: %d / %d", attempts, maxGuesses));
    }

    // UI niceties
    private void flashLabel(JLabel label, String text) {
        Color original = label.getForeground();
        label.setText(text);
        label.setForeground(Color.RED.darker());
        Timer t = new Timer(900, e -> label.setForeground(original));
        t.setRepeats(false);
        t.start();
    }

    private void flashBackground(Color c) {
        Color original = getContentPane().getBackground();
        getContentPane().setBackground(c);
        Timer t = new Timer(700, e -> getContentPane().setBackground(original));
        t.setRepeats(false);
        t.start();
    }

    // High score persistence
    private void loadHighScores() {
        Path p = Path.of(HS_FILE);
        if (Files.exists(p)) {
            try (InputStream in = Files.newInputStream(p)) {
                highscores.load(in);
            } catch (IOException ignored) {}
        }
    }

    private void saveHighScores() {
        try (OutputStream out = Files.newOutputStream(Path.of(HS_FILE))) {
            highscores.store(out, "GuessTheNumber High Scores: best attempts (lower is better)");
        } catch (IOException ignored) {}
    }

    private void updateHighScoreIfNeeded() {
        String key = difficultyKey();
        String current = highscores.getProperty(key);
        if (current == null) {
            highscores.setProperty(key, String.valueOf(attempts));
            saveHighScores();
            updateHighScoreLabel();
            return;
        }
        try {
            int best = Integer.parseInt(current);
            if (attempts < best) {
                highscores.setProperty(key, String.valueOf(attempts));
                saveHighScores();
            }
        } catch (NumberFormatException ignored) {}
        updateHighScoreLabel();
    }

    private String difficultyKey() {
        return switch (difficultyCombo.getSelectedIndex()) {
            case 0 -> "easy";
            case 2 -> "hard";
            default -> "medium";
        };
    }

    private void updateHighScoreLabel() {
        String e = highscores.getProperty("easy", "N/A");
        String m = highscores.getProperty("medium", "N/A");
        String h = highscores.getProperty("hard", "N/A");
        highScoreLabel.setText(String.format("High Scores — Easy: %s  Medium: %s  Hard: %s", e, m, h));
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(() -> {
            try {
                // Nice cross-platform look
                UIManager.setLookAndFeel(UIManager.getSystemLookAndFeelClassName());
            } catch (Exception ignored) {}
            GuessTheNumberSwing frame = new GuessTheNumberSwing();
            frame.setVisible(true);
        });
    }
}
```
---
# 🎲 Guess The Number — Swing Edition

A feature-rich, graphical user interface (GUI) version of the classic "Guess the Number" game, built using **Java Swing**.

This application goes beyond the basic console game, offering a polished experience with adjustable difficulty, smart hints, attempt limits, and persistent high scores.

## ✨ Features

* **Difficulty Settings:** Choose from three modes that adjust the guessing range and the maximum number of attempts:
    * **Easy:** 1 - 20 (fewer attempts)
    * **Medium:** 1 - 100
    * **Hard:** 1 - 1000 (more attempts, bigger challenge)
* **Intelligent Hints:** Receive standard "Too High/Too Low" feedback, plus **proximity hints** ("🔥 Burning Hot," "♨️ Warm," "🥶 Cold") to guide your next guess.
* **Attempt Limits:** The maximum number of guesses is calculated dynamically based on the difficulty range (approximating $\log_2(\text{max})$) to ensure a fair but challenging experience.
* **High Score Persistence:** Your best score (lowest number of attempts) for each difficulty level is automatically saved to a local file (`guess_highscores.properties`) and loaded every time you start the game.
* **User-Friendly Interface:**
    * Clean, responsive Swing layout.
    * Enter key in the guess field submits the guess.
    * "New Game" and "Exit" options available via a top menu.
    * Visual cues (color flashes) for wins and losses.

***

## 🚀 Getting Started

### Prerequisites

You need to have the **Java Development Kit (JDK) version 8 or later** installed on your system.

### Running the Application

Since the entire application is contained within a single `.java` file, running it is straightforward:

1.  **Save:** Save the provided code into a file named `GuessTheNumberSwing.java`.
2.  **Compile:** Open your terminal or command prompt in the directory where you saved the file and compile it:
    ```bash
    javac GuessTheNumberSwing.java
    ```
3.  **Run:** Execute the compiled class file:
    ```bash
    java GuessTheNumberSwing
    ```

The GUI window should immediately appear, ready for you to select a difficulty and start playing!

***

## ⚙️ Game Logic Highlights (For Developers)

* **Difficulty Calculation:** The maximum number of guesses (`maxGuesses`) is calculated using the formula:
    $$\text{maxGuesses} = \text{ceil}(\log_2(\text{max\_range})) + 1$$
    This is based on the optimal strategy of **binary search**, ensuring that a perfect player has enough attempts to guarantee a win.
* **High Score Storage:** The application uses the built-in Java `java.util.Properties` class to manage high scores. Scores are saved to a file named `guess_highscores.properties` in the application directory.
* **Keyboard Shortcuts:** Pressing **`Ctrl + N`** (or equivalent on some systems) immediately starts a new game, mirroring standard desktop application behavior.
* **`UIManager.getSystemLookAndFeelClassName()`:** This line ensures the application uses the native look and feel of the operating system (e.g., Windows, macOS, Linux), making it look integrated and modern.
