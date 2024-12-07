# crispy-goggles

runtimeScene.setBackgroundColor(100,100,240);

// Configurazione del labirinto
const rows = 31;
const cols = 31;
const cellSize = 64;

// Inizializza il labirinto pieno di muri
const maze = Array.from({ length: rows }, () => Array(cols).fill(1));

// Funzione per generare corridoi vuoti garantiti
function addGuaranteedCorridors(maze, numVertical, numHorizontal) {
    const verticalCorridors = new Set();
    const horizontalCorridors = new Set();

    while (verticalCorridors.size < numVertical) {
        const x = Math.floor(Math.random() * ((cols - 1) / 2)) * 2 + 1;
        if (!verticalCorridors.has(x)) {
            verticalCorridors.add(x);
            for (let y = 1; y < rows - 1; y++) {
                maze[y][x] = 0;
            }
        }
    }

    while (horizontalCorridors.size < numHorizontal) {
        const y = Math.floor(Math.random() * ((rows - 1) / 2)) * 2 + 1;
        if (!horizontalCorridors.has(y)) {
            horizontalCorridors.add(y);
            for (let x = 1; x < cols - 1; x++) {
                maze[y][x] = 0;
            }
        }
    }
}

// Funzione per generare il labirinto usando DFS
function generateMazeDFS(maze) {
    const stack = [];
    const visited = Array.from({ length: rows }, () => Array(cols).fill(false));
    let currentCell = { x: 1, y: 1 };
    maze[currentCell.y][currentCell.x] = 0;
    visited[currentCell.y][currentCell.x] = true;
    stack.push(currentCell);

    const directions = [
        { dx: 0, dy: -2 },
        { dx: 2, dy: 0 },
        { dx: 0, dy: 2 },
        { dx: -2, dy: 0 }
    ];

    while (stack.length > 0) {
        const { x, y } = currentCell;
        const possibleMoves = [];

        for (const { dx, dy } of directions) {
            const nx = x + dx;
            const ny = y + dy;
            if (
                nx > 0 && ny > 0 && nx < cols - 1 && ny < rows - 1 &&
                !visited[ny][nx]
            ) {
                possibleMoves.push({ x: nx, y: ny, wallX: x + dx / 2, wallY: y + dy / 2 });
            }
        }

        if (possibleMoves.length > 0) {
            const nextMove = possibleMoves[Math.floor(Math.random() * possibleMoves.length)];
            maze[nextMove.wallY][nextMove.wallX] = 0;
            maze[nextMove.y][nextMove.x] = 0;
            visited[nextMove.y][nextMove.x] = true;

            stack.push(currentCell);
            currentCell = { x: nextMove.x, y: nextMove.y };
        } else {
            currentCell = stack.pop();
        }
    }
}

// Funzione per creare la ghost house
function createGhostHouse(maze) {
    const centerX = Math.floor(cols / 2);
    const centerY = Math.floor(rows / 2);
    const width = 5;
    const height = 5;
    
    // Assicurati che ci sia un corridoio di accesso sopra la ghost house
    for (let y = centerY - Math.floor(height/2) - 1; y >= centerY - Math.floor(height/2) - 2; y--) {
        maze[y][centerX] = 0;
    }
    
    // Crea la ghost house
    for (let y = centerY - Math.floor(height/2); y <= centerY + Math.floor(height/2); y++) {
        for (let x = centerX - Math.floor(width/2); x <= centerX + Math.floor(width/2); x++) {
            if (x === centerX - Math.floor(width/2) || 
                x === centerX + Math.floor(width/2) ||
                y === centerY - Math.floor(height/2) || 
                y === centerY + Math.floor(height/2)) {
                maze[y][x] = 2; // muri della ghost house
            } else {
                maze[y][x] = 3; // interno della ghost house
            }
        }
    }
    
    // Assicurati che l'entrata sia sempre libera
    maze[centerY - Math.floor(height/2)][centerX] = 3; // entrata
    
    return {
        centerX: centerX,
        centerY: centerY,
        width: width,
        height: height
    };
}

// Funzione per trovare posizione valida per il giocatore
function findStartPosition(maze) {
    for (let y = 1; y < rows - 1; y++) {
        for (let x = 1; x < cols - 1; x++) {
            if (maze[y][x] === 0) {
                if (
                    (maze[y - 1][x] === 0 || maze[y + 1][x] === 0 ||
                     maze[y][x - 1] === 0 || maze[y][x + 1] === 0)
                ) {
                    return { x, y };
                }
            }
        }
    }
    return null;
}

// Generate maze and ghost house
generateMazeDFS(maze);
addGuaranteedCorridors(maze, 4, 4);
const ghostHouse = createGhostHouse(maze);

// Draw walls
maze.forEach((row, y) => {
    row.forEach((cell, x) => {
        if (cell === 1 || cell === 2) {
            const wall = runtimeScene.createObject("Wall");
            wall.setPosition(x * cellSize, y * cellSize);
            wall.activateBehavior("PathfindingObstacle", true);
        }
    });
});

// Place player and pellets
const startPosition = findStartPosition(maze);
if (startPosition) {
    const player = runtimeScene.createObject("Player");
    player.setPosition(startPosition.x * cellSize, startPosition.y * cellSize);

    // Place pellets
    for (let y = 1; y < rows - 1; y++) {
        for (let x = 1; x < cols - 1; x++) {
            if (maze[y][x] === 0) {  // Se la cella è vuota
                const distanceFromStart = Math.sqrt(
                    Math.pow(x - startPosition.x, 2) + 
                    Math.pow(y - startPosition.y, 2)
                );
                
                if (distanceFromStart > 2 && maze[y][x] !== 3) {
                    const pelletX = (x * cellSize) + (cellSize * 0.5);
                    const pelletY = (y * cellSize) + (cellSize * 0.5);

                    if (Math.random() < 0.05) {
                        const powerPellet = runtimeScene.createObject("PowerPellet");
                        if (powerPellet) {
                            powerPellet.setPosition(pelletX, pelletY);
                        }
                    } else {
                        const pellet = runtimeScene.createObject("Pellet");
                        if (pellet) {
                            pellet.setPosition(pelletX, pelletY);
                        }
                    }
                }
            }
        }
    }
} else {
    console.error("No valid position found for player");
}

// Place ghosts
const numEnemies = 4;
const enemySpacing = cellSize * 1.5;

for (let i = 0; i < numEnemies; i++) {
    const enemy = runtimeScene.createObject("Enemy");
    const offsetX = (i - (numEnemies-1)/2) * enemySpacing;
    enemy.setPosition(
        ghostHouse.centerX * cellSize + offsetX,
        ghostHouse.centerY * cellSize
    );
}

// Initialize game variables
runtimeScene.getVariables().get("Score").setNumber(0);
runtimeScene.getVariables().get("PowerMode").setBoolean(false);
runtimeScene.getVariables().get("PowerModeTimer").setNumber(0);

// Debug: count walls
const walls = runtimeScene.getObjects("Wall");
console.log("Number of walls generated:", walls.length);