# ComputerProgrammingFinalProject
package com.example.demo;

import javafx.animation.AnimationTimer;
import javafx.animation.KeyFrame;
import javafx.animation.Timeline;
import javafx.application.Application;
import javafx.geometry.Rectangle2D;
import javafx.scene.Scene;
import javafx.scene.input.KeyCode;
import javafx.scene.layout.Pane;
import javafx.scene.paint.Color;
import javafx.scene.shape.Rectangle;
import javafx.scene.text.Font;
import javafx.scene.text.Text;
import javafx.stage.Stage;
import javafx.scene.image.Image;
import javafx.scene.image.ImageView;
import javafx.util.Duration;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.Random;
import java.net.URL;

public class HelloApplication extends Application {

    private static final int WIDTH = 400;
    private static final int HEIGHT = 600;
    private static final int PIPE_WIDTH = 60;
    private static final int GAP = 150;
    private static final int PIPE_SPEED = 3;

    private ImageView birdView;
    private double velocity = 0;
    private final double gravity = 0.5;
    private final double jumpStrength = -8;
    private final double maxFallSpeed = 10;

    private Pane root;
    private ArrayList<Rectangle> pipes = new ArrayList<>();
    private ArrayList<Boolean> scoredPipes = new ArrayList<>();
    private AnimationTimer timer;
    private int frameCount = 0;

    private boolean gameOver = false;
    private boolean gameStarted = false;
    private int score = 0;

    private Text scoreText;
    private Text messageText;

    private final double birdWidth = 60;
    private final double birdHeight = 60;

    @Override
    public void start(Stage primaryStage) {
        root = new Pane();

        // Load and add background image
        URL bgURL = getClass().getResource("/background.png");
        if (bgURL == null) {
            throw new RuntimeException("background.png not found in resources!");
        }
        Image backgroundImage = new Image(bgURL.toExternalForm());
        ImageView backgroundView = new ImageView(backgroundImage);
        backgroundView.setFitWidth(WIDTH);
        backgroundView.setFitHeight(HEIGHT);
        backgroundView.setPreserveRatio(false);
        root.getChildren().add(backgroundView);

        Scene scene = new Scene(root, WIDTH, HEIGHT);

        // Load bird sprite (1536x1024, 3 frames of 512x1024)
        URL birdURL = getClass().getResource("/bird.png");
        if (birdURL == null) {
            throw new RuntimeException("bird.png not found in resources!");
        }

        Image birdImage = new Image(birdURL.toExternalForm());
        birdView = new ImageView(birdImage);
        birdView.setFitWidth(birdWidth);
        birdView.setFitHeight(birdHeight);
        birdView.setPreserveRatio(false);
        birdView.setSmooth(true);
        birdView.setX(100); // Fixed horizontal position
        birdView.setY(HEIGHT / 2 - birdHeight / 2);
        birdView.setViewport(new Rectangle2D(0, 0, 512, 1024)); // first frame

        // Flap animation - cycle through 3 frames
        final int[] frameIndex = {0};
        Timeline flapAnimation = new Timeline(
                new KeyFrame(Duration.millis(100), e -> {
                    frameIndex[0] = (frameIndex[0] + 1) % 3;
                    birdView.setViewport(new Rectangle2D(frameIndex[0] * 512, 0, 512, 1024));
                })
        );
        flapAnimation.setCycleCount(Timeline.INDEFINITE);
        flapAnimation.play();

        root.getChildren().add(birdView);

        scoreText = new Text("Score: 0");
        scoreText.setFont(Font.font(24));
        scoreText.setFill(Color.BLACK);
        scoreText.setX(10);
        scoreText.setY(30);
        root.getChildren().add(scoreText);

        messageText = new Text("Press SPACE to Start");
        messageText.setFont(Font.font(24));
        messageText.setFill(Color.BLUE);
        messageText.setX(WIDTH / 2 - 100);
        messageText.setY(HEIGHT / 2);
        root.getChildren().add(messageText);

        scene.setOnKeyPressed(e -> {
            if (!gameStarted && e.getCode() == KeyCode.SPACE) {
                gameStarted = true;
                messageText.setText("");
            } else if (gameStarted && !gameOver && e.getCode() == KeyCode.SPACE) {
                velocity = jumpStrength;
            } else if (gameOver && e.getCode() == KeyCode.R) {
                restartGame();
            }
        });

        timer = new AnimationTimer() {
            @Override
            public void handle(long now) {
                update();
            }
        };
        timer.start();

        primaryStage.setTitle("Flappy Bird JavaFX");
        primaryStage.setScene(scene);
        primaryStage.show();
    }

    private void update() {
        if (!gameStarted || gameOver) return;

        velocity += gravity;
        if (velocity > maxFallSpeed) velocity = maxFallSpeed;
        birdView.setY(birdView.getY() + velocity);

        if (frameCount % 100 == 0) {
            addPipe();
        }

        movePipes();
        checkCollision();
        checkScore();

        if (birdView.getY() + birdHeight > HEIGHT || birdView.getY() < 0) {
            endGame();
        }

        frameCount++;
    }

    private void addPipe() {
        Random rand = new Random();
        int gapStart = rand.nextInt(HEIGHT - GAP - 200) + 100;

        Rectangle top = new Rectangle(PIPE_WIDTH, gapStart);
        top.setX(WIDTH);
        top.setY(0);
        top.setFill(Color.GREEN);

        Rectangle bottom = new Rectangle(PIPE_WIDTH, HEIGHT - gapStart - GAP);
        bottom.setX(WIDTH);
        bottom.setY(gapStart + GAP);
        bottom.setFill(Color.GREEN);

        pipes.add(top);
        pipes.add(bottom);
        scoredPipes.add(false);
        root.getChildren().addAll(top, bottom);
    }

    private void movePipes() {
        for (Rectangle pipe : pipes) {
            pipe.setX(pipe.getX() - PIPE_SPEED);
        }
    }

    private void checkCollision() {
        // Use custom hitbox to match the visual bird size
        double bx = birdView.getX();
        double by = birdView.getY();
        double bw = birdView.getFitWidth();
        double bh = birdView.getFitHeight();

        Rectangle2D birdHitbox = new Rectangle2D(bx + 10, by + 10, bw - 20, bh - 20);

        for (Rectangle pipe : pipes) {
            Rectangle2D pipeBounds = new Rectangle2D(
                    pipe.getX(), pipe.getY(), pipe.getWidth(), pipe.getHeight());

            if (pipeBounds.intersects(birdHitbox)) {
                endGame();
                break;
            }
        }
    }

    private void checkScore() {
        for (int i = 0; i < pipes.size(); i += 2) {
            Rectangle topPipe = pipes.get(i);
            boolean scored = scoredPipes.get(i / 2);
            if (!scored && topPipe.getX() + PIPE_WIDTH < birdView.getX()) {
                score++;
                scoreText.setText("Score: " + score);
                scoredPipes.set(i / 2, true);
            }
        }
    }

    private void endGame() {
        gameOver = true;
        messageText.setText("Game Over! Score: " + score + "\nPress R to Restart");
    }

    private void restartGame() {
        root.getChildren().removeIf(node -> node instanceof Rectangle);
        pipes.clear();
        scoredPipes.clear();
        birdView.setY(HEIGHT / 2 - birdHeight / 2);
        velocity = 0;
        frameCount = 0;
        score = 0; 
        scoreText.setText("Score: 0");
        gameOver = false;
        gameStarted = false;
        messageText.setText("Press SPACE to Start");
    }

    public static void main(String[] args) {
        launch(args);
    }
}
