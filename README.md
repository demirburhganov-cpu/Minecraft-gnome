#include "raylib.h"
#include "raymath.h"
#include "rlgl.h"
#include <string>
#include <vector>
#include <cmath>
#include <algorithm>

#define SADDLEBROWN   (Color){ 139, 69, 19, 255 }
#define CHOCOLATE     (Color){ 210, 105, 30, 255 }
#define STONE_GRAY    (Color){ 110, 110, 110, 255 }
#define DIAMOND_CYAN  (Color){ 0, 229, 238, 255 }
#define BEDROCK_DARK  (Color){ 40, 40, 40, 255 }

struct Block {
    Vector3 position;
    int type;      // 1: Трава, 2: Земля, 3: Камень, 4: Алмаз, 5: Бедрок, 6: Дерево, 7: Листва
    Color color;
};

struct InventorySlot {
    int blockType;
    int count;
    Color color;
};

// В новых версиях raylib (5.5+) DrawCubeTexture убрана из библиотеки,
// поэтому рисуем куб с текстурой на всех 6 гранях сами.
void DrawTexturedCube(Texture2D texture, Vector3 position, float width, float height, float length, Color color) {
    float x = position.x;
    float y = position.y;
    float z = position.z;
    float hw = width / 2.0f;
    float hh = height / 2.0f;
    float hl = length / 2.0f;

    rlCheckRenderBatchLimit(24);
    rlSetTexture(texture.id);

    rlBegin(RL_QUADS);
        rlColor4ub(color.r, color.g, color.b, color.a);

        // Передняя грань
        rlNormal3f(0.0f, 0.0f, 1.0f);
        rlTexCoord2f(0.0f, 0.0f); rlVertex3f(x - hw, y - hh, z + hl);
        rlTexCoord2f(1.0f, 0.0f); rlVertex3f(x + hw, y - hh, z + hl);
        rlTexCoord2f(1.0f, 1.0f); rlVertex3f(x + hw, y + hh, z + hl);
        rlTexCoord2f(0.0f, 1.0f); rlVertex3f(x - hw, y + hh, z + hl);

        // Задняя грань
        rlNormal3f(0.0f, 0.0f, -1.0f);
        rlTexCoord2f(1.0f, 0.0f); rlVertex3f(x - hw, y - hh, z - hl);
        rlTexCoord2f(1.0f, 1.0f); rlVertex3f(x - hw, y + hh, z - hl);
        rlTexCoord2f(0.0f, 1.0f); rlVertex3f(x + hw, y + hh, z - hl);
        rlTexCoord2f(0.0f, 0.0f); rlVertex3f(x + hw, y - hh, z - hl);

        // Верхняя грань
        rlNormal3f(0.0f, 1.0f, 0.0f);
        rlTexCoord2f(0.0f, 1.0f); rlVertex3f(x - hw, y + hh, z - hl);
        rlTexCoord2f(0.0f, 0.0f); rlVertex3f(x - hw, y + hh, z + hl);
        rlTexCoord2f(1.0f, 0.0f); rlVertex3f(x + hw, y + hh, z + hl);
        rlTexCoord2f(1.0f, 1.0f); rlVertex3f(x + hw, y + hh, z - hl);

        // Нижняя грань
        rlNormal3f(0.0f, -1.0f, 0.0f);
        rlTexCoord2f(1.0f, 1.0f); rlVertex3f(x - hw, y - hh, z - hl);
        rlTexCoord2f(0.0f, 1.0f); rlVertex3f(x + hw, y - hh, z - hl);
        rlTexCoord2f(0.0f, 0.0f); rlVertex3f(x + hw, y - hh, z + hl);
        rlTexCoord2f(1.0f, 0.0f); rlVertex3f(x - hw, y - hh, z + hl);

        // Правая грань
        rlNormal3f(1.0f, 0.0f, 0.0f);
        rlTexCoord2f(1.0f, 0.0f); rlVertex3f(x + hw, y - hh, z - hl);
        rlTexCoord2f(1.0f, 1.0f); rlVertex3f(x + hw, y + hh, z - hl);
        rlTexCoord2f(0.0f, 1.0f); rlVertex3f(x + hw, y + hh, z + hl);
        rlTexCoord2f(0.0f, 0.0f); rlVertex3f(x + hw, y - hh, z + hl);

        // Левая грань
        rlNormal3f(-1.0f, 0.0f, 0.0f);
        rlTexCoord2f(0.0f, 0.0f); rlVertex3f(x - hw, y - hh, z - hl);
        rlTexCoord2f(1.0f, 0.0f); rlVertex3f(x - hw, y - hh, z + hl);
        rlTexCoord2f(1.0f, 1.0f); rlVertex3f(x - hw, y + hh, z + hl);
        rlTexCoord2f(0.0f, 1.0f); rlVertex3f(x - hw, y + hh, z - hl);
    rlEnd();

    rlSetTexture(0);
}

int main() {
    const int screenWidth = 1280;
    const int screenHeight = 720;
    InitWindow(screenWidth, screenHeight, "Minecraft Torrent Edition - No Fall Through");

    Camera3D camera = { 0 };
    camera.position = (Vector3){ 0.0f, 15.0f, 0.0f };
    camera.target = (Vector3){ 0.0f, 15.0f, 1.0f };
    camera.up = (Vector3){ 0.0f, 1.0f, 0.0f };
    camera.fovy = 60.0f;
    camera.projection = CAMERA_PERSPECTIVE;

    DisableCursor();
    SetTargetFPS(60);

    // ТЕКСТУРЫ БЛОКОВ
    // Папка assets лежит рядом с exe. Одна и та же картинка рисуется на всех 6 гранях куба.
    // Если файла нет — блок рисуется обычным цветом.
    Texture2D blockTex[8] = {};
    bool hasTex[8] = { false };
    const char* texFiles[8] = {
        nullptr,
        "assets/grass.png",               // 1 трава
        "assets/dirt.png",                // 2 земля
        "assets/stone.png",               // 3 камень
        "assets/diamond.png",             // 4 алмаз
        "assets/nokia-3310-orignal.png",  // 5 бедрок
        "assets/wood.png",                // 6 дерево
        "assets/leaves.png"               // 7 листва
    };
    for (int i = 1; i < 8; i++) {
        if (texFiles[i] && FileExists(texFiles[i])) {
            blockTex[i] = LoadTexture(texFiles[i]);
            hasTex[i] = true;
        }
    }

    bool inMenu = true;
    bool inventoryOpen = false;
    bool isFlying = false;
    int selectedSlot = 0;

    InventorySlot hotbar[9] = {
        {1, 64, GREEN},        // Grass
        {2, 64, CHOCOLATE},    // Dirt
        {3, 64, STONE_GRAY},   // Stone
        {4, 16, DIAMOND_CYAN}, // Diamond
        {6, 32, SADDLEBROWN},  // Wood
        {7, 32, DARKGREEN},    // Leaves
        {0, 0, BLANK},
        {0, 0, BLANK},
        {0, 0, BLANK}
    };

    float verticalVelocity = 0.0f;
    const float gravity = 0.03f;
    const float jumpForce = 0.42f;
    bool isGrounded = false;

    // Двигаем камеру по вертикали вместе с точкой взгляда (иначе при прыжке взгляд "клюёт" вниз)
    auto moveCameraY = [&](float dy) {
        camera.position.y += dy;
        camera.target.y += dy;
    };

    // Иконка блока в интерфейсе (текстура или цвет)
    auto drawIcon = [&](int type, Color col, int x, int y, int size) {
        if (type > 0 && type < 8 && hasTex[type]) {
            Rectangle src = { 0.0f, 0.0f, (float)blockTex[type].width, (float)blockTex[type].height };
            Rectangle dst = { (float)x, (float)y, (float)size, (float)size };
            DrawTexturePro(blockTex[type], src, dst, Vector2{ 0.0f, 0.0f }, 0.0f, WHITE);
        } else {
            DrawRectangle(x, y, size, size, col);
        }
    };

    // ГЕНЕРАЦИЯ МИРА
    std::vector<Block> worldBlocks;
    int mapRadius = 35;

    for (int x = -mapRadius; x <= mapRadius; x++) {
        for (int z = -mapRadius; z <= mapRadius; z++) {
            float mountainHeight = sinf(x * 0.2f) * cosf(z * 0.2f) * 6.0f + cosf(x * 0.1f) * 4.0f;
            int groundLevel = (int)(5.0f + mountainHeight);
            if (groundLevel < 2) groundLevel = 2;

            for (int y = 0; y <= groundLevel; y++) {
                int blockType = (y < groundLevel - 2) ? 3 : (y < groundLevel ? 2 : 1);

                // Нижний слой мира — бедрок
                if (y == 0) blockType = 5;
                
                bool isCave = (y > 1 && y < groundLevel - 1 && (x * 7 + y * 13 + z * 3) % 11 == 0);
                if (isCave) {
                    if ((x * 2 + y * 5 + z * 3) % 17 == 0) {
                        blockType = 4;
                    } else {
                        continue;
                    }
                }

                Color blockColor = (blockType == 5) ? BEDROCK_DARK : ((blockType == 4) ? DIAMOND_CYAN : ((blockType == 3) ? STONE_GRAY : ((blockType == 2) ? CHOCOLATE : GREEN)));
                worldBlocks.push_back({ (Vector3) { (float)x, (float)y, (float)z }, blockType, blockColor });
            }

            // Деревья
            if ((x * 3 + z * 7) % 9 == 0 && x > -(mapRadius-3) && x < (mapRadius-3) && z > -(mapRadius-3) && z < (mapRadius-3)) {
                int treeBaseY = groundLevel + 1;
                int trunkHeight = 4 + ((x + z) % 3);

                for (int ty = 0; ty < trunkHeight; ty++) {
                    worldBlocks.push_back({ (Vector3) { (float)x, (float)(treeBaseY + ty), (float)z }, 6, SADDLEBROWN });
                }

                for (int lx = -1; lx <= 1; lx++) {
                    for (int lz = -1; lz <= 1; lz++) {
                        for (int ly = 0; ly < 2; ly++) {
                            worldBlocks.push_back({ (Vector3) { (float)(x + lx), (float)(treeBaseY + trunkHeight + ly), (float)(z + lz) }, 7, DARKGREEN });
                        }
                    }
                }
            }
        }
    }

    while (!WindowShouldClose()) {
        float wheel = GetMouseWheelMove();
        if (wheel != 0 && !inventoryOpen) {
            selectedSlot -= (int)wheel;
            if (selectedSlot < 0) selectedSlot = 8;
            if (selectedSlot > 8) selectedSlot = 0;
        }
        for (int i = 0; i < 9; i++) {
            if (IsKeyPressed(KEY_ONE + i) && !inventoryOpen) selectedSlot = i;
        }

        if (IsKeyPressed(KEY_M)) {
            inMenu = !inMenu;
            if (inMenu || inventoryOpen) EnableCursor();
            else DisableCursor();
        }

        if (!inMenu) {
            if (IsKeyPressed(KEY_E)) {
                inventoryOpen = !inventoryOpen;
                if (inventoryOpen) EnableCursor();
                else DisableCursor();
            }

            if (!inventoryOpen) {
                UpdateCamera(&camera, CAMERA_FIRST_PERSON);

                if (IsKeyPressed(KEY_R)) {
                    isFlying = !isFlying;
                    verticalVelocity = 0.0f;
                }

                // Горизонтальные границы карты, чтобы не уйти в бесконечность
                float limit = (float)mapRadius - 1.0f;
                if (camera.position.x < -limit) camera.position.x = -limit;
                if (camera.position.x > limit) camera.position.x = limit;
                if (camera.position.z < -limit) camera.position.z = -limit;
                if (camera.position.z > limit) camera.position.z = limit;

                // Защита: если провалились глубоко под мир (ниже -5), возвращаем на спавн
                if (camera.position.y < -5.0f) {
                    camera.position = (Vector3){ 0.0f, 15.0f, 0.0f };
                    camera.target = (Vector3){ 0.0f, 15.0f, 1.0f };
                    verticalVelocity = 0.0f;
                }

                if (!isFlying) {
                    const float eyeHeight = 1.5f;     // расстояние от глаз до ног
                    const float playerHalf = 0.3f;    // половина ширины игрока

                    verticalVelocity -= gravity;
                    if (verticalVelocity < -0.5f) verticalVelocity = -0.5f;

                    float prevFeetY = camera.position.y - eyeHeight;
                    moveCameraY(verticalVelocity);
                    float newFeetY = camera.position.y - eyeHeight;

                    // Ищем самый высокий блок под ногами, через верх которого игрок прошёл за этот кадр
                    float floorTop = -1000.0f;
                    bool foundFloor = false;

                    if (verticalVelocity <= 0.0f) {
                        for (const auto& b : worldBlocks) {
                            if (fabsf(camera.position.x - b.position.x) < 0.5f + playerHalf &&
                                fabsf(camera.position.z - b.position.z) < 0.5f + playerHalf) {
                                float blockTop = b.position.y + 0.5f;
                                if (prevFeetY >= blockTop - 0.01f && newFeetY <= blockTop && blockTop > floorTop) {
                                    floorTop = blockTop;
                                    foundFloor = true;
                                }
                            }
                        }
                    }

                    if (foundFloor) {
                        moveCameraY(floorTop + eyeHeight - camera.position.y);
                        verticalVelocity = 0.0f;
                        isGrounded = true;
                    } else {
                        isGrounded = false;
                    }

                    if (IsKeyPressed(KEY_SPACE) && isGrounded) {
                        verticalVelocity = jumpForce;
                        isGrounded = false;
                    }
                } else {
                    if (IsKeyDown(KEY_SPACE)) moveCameraY(0.25f);
                    if (IsKeyDown(KEY_LEFT_CONTROL)) moveCameraY(-0.25f);
                }

                // ЛЕВАЯ КНОПКА МЫШИ — ЛОМАТЬ
                if (IsMouseButtonPressed(MOUSE_BUTTON_LEFT)) {
                    int hitIndex = -1;
                    float minDistance = 10.0f;
                    Ray ray = { camera.position, Vector3Normalize(Vector3Subtract(camera.target, camera.position)) };

                    for (size_t i = 0; i < worldBlocks.size(); i++) {
                        BoundingBox box = {
                            { worldBlocks[i].position.x - 0.5f, worldBlocks[i].position.y - 0.5f, worldBlocks[i].position.z - 0.5f },
                            { worldBlocks[i].position.x + 0.5f, worldBlocks[i].position.y + 0.5f, worldBlocks[i].position.z + 0.5f }
                        };
                        RayCollision collision = GetRayCollisionBox(ray, box);
                        if (collision.hit && collision.distance < minDistance) {
                            minDistance = collision.distance;
                            hitIndex = (int)i;
                        }
                    }

                    // Бедрок сломать нельзя
                    if (hitIndex != -1 && worldBlocks[hitIndex].type != 5) {
                        int brokenType = worldBlocks[hitIndex].type;
                        Color brokenColor = worldBlocks[hitIndex].color;

                        worldBlocks.erase(worldBlocks.begin() + hitIndex);

                        if (hotbar[selectedSlot].blockType == 0 || hotbar[selectedSlot].blockType == brokenType) {
                            hotbar[selectedSlot].blockType = brokenType;
                            hotbar[selectedSlot].count++;
                            hotbar[selectedSlot].color = brokenColor;
                        } else {
                            for (int i = 0; i < 9; i++) {
                                if (hotbar[i].blockType == 0 || hotbar[i].blockType == brokenType) {
                                    hotbar[i].blockType = brokenType;
                                    hotbar[i].count++;
                                    hotbar[i].color = brokenColor;
                                    break;
                                }
                            }
                        }
                    }
                }

                // ПРАВАЯ КНОПКА МЫШИ — СТРОИТЬ
                if (IsMouseButtonPressed(MOUSE_BUTTON_RIGHT)) {
                    if (hotbar[selectedSlot].blockType != 0 && hotbar[selectedSlot].count > 0) {
                        Vector3 placePos = {
                            roundf(camera.position.x + (camera.target.x - camera.position.x) * 0.2f),
                            roundf(camera.position.y + (camera.target.y - camera.position.y) * 0.2f),
                            roundf(camera.position.z + (camera.target.z - camera.position.z) * 0.2f)
                        };
                        worldBlocks.push_back({ placePos, hotbar[selectedSlot].blockType, hotbar[selectedSlot].color });
                        
                        hotbar[selectedSlot].count--;
                        if (hotbar[selectedSlot].count <= 0) {
                            hotbar[selectedSlot].blockType = 0;
                        }
                    }
                }
            }
        }

        BeginDrawing();
        ClearBackground(SKYBLUE);

        if (inMenu) {
            DrawRectangle(0, 0, screenWidth, screenHeight, Fade(BLACK, 0.9f));
            DrawText("MINECRAFT TORRENT EDITION", screenWidth / 2 - 220, 100, 30, RAYWHITE);
            DrawText("No fall-through bug! Jump as high as you want.", screenWidth / 2 - 200, 150, 14, LIME);

            DrawRectangle(screenWidth / 2 - 150, 240, 300, 50, DARKGRAY);
            DrawText("Press [M] to Play", screenWidth / 2 - 70, 255, 16, WHITE);
        } else {
            BeginMode3D(camera);

            // Блоки рисуем по типам, чтобы текстуры не переключались на каждом кубе
            for (int t = 1; t < 8; t++) {
                for (const auto& b : worldBlocks) {
                    if (b.type != t) continue;
                    if (hasTex[t]) DrawTexturedCube(blockTex[t], b.position, 1.0f, 1.0f, 1.0f, WHITE);
                    else DrawCube(b.position, 1.0f, 1.0f, 1.0f, b.color);
                }
            }

            EndMode3D();

            DrawText("FPS:", 10, 10, 14, WHITE);
            DrawText(TextFormat("%d", GetFPS()), 45, 10, 14, LIME);
            DrawText("LMB: Break | RMB: Place | [E] Inventory | [R] Fly", 10, 30, 14, YELLOW);

            if (inventoryOpen) {
                DrawRectangle(0, 0, screenWidth, screenHeight, Fade(BLACK, 0.7f));
                DrawRectangle(screenWidth / 2 - 200, screenHeight / 2 - 150, 400, 300, DARKGRAY);
                DrawRectangleLines(screenWidth / 2 - 200, screenHeight / 2 - 150, 400, 300, WHITE);
                
                DrawText("INVENTORY", screenWidth / 2 - 50, screenHeight / 2 - 130, 20, RAYWHITE);
                DrawText("Press [E] to close", screenWidth / 2 - 65, screenHeight / 2 + 110, 12, LIGHTGRAY);

                for (int i = 0; i < 9; i++) {
                    int ix = (i % 3) - 1;
                    int iy = (i / 3) - 1;
                    int slotX = screenWidth / 2 + ix * 60 - 25;
                    int slotY = screenHeight / 2 + iy * 60 - 25;

                    Color slotCol = (i == selectedSlot) ? YELLOW : BLACK;
                    DrawRectangle(slotX, slotY, 50, 50, slotCol);
                    DrawRectangleLines(slotX, slotY, 50, 50, WHITE);

                    if (hotbar[i].blockType != 0 && hotbar[i].count > 0) {
                        drawIcon(hotbar[i].blockType, hotbar[i].color, slotX + 10, slotY + 10, 30);
                        DrawText(TextFormat("%d", hotbar[i].count), slotX + 25, slotY + 30, 10, WHITE);
                    }
                }
            } else {
                int hotbarWidth = 9 * 50;
                int hotbarX = (screenWidth - hotbarWidth) / 2;
                int hotbarY = screenHeight - 70;

                DrawRectangle(hotbarX - 10, hotbarY - 10, hotbarWidth + 20, 70, Fade(DARKGRAY, 0.85f));
                for (int i = 0; i < 9; i++) {
                    Color slotColor = (i == selectedSlot) ? YELLOW : BLACK;
                    DrawRectangle(hotbarX + (i * 50), hotbarY, 45, 45, slotColor);
                    DrawRectangleLines(hotbarX + (i * 50), hotbarY, 45, 45, WHITE);

                    if (hotbar[i].blockType != 0 && hotbar[i].count > 0) {
                        drawIcon(hotbar[i].blockType, hotbar[i].color, hotbarX + (i * 50) + 8, hotbarY + 8, 29);
                        DrawText(TextFormat("%d", hotbar[i].count), hotbarX + (i * 50) + 22, hotbarY + 28, 10, RAYWHITE);
                    }
                    DrawText(TextFormat("%d", i + 1), hotbarX + (i * 50) + 4, hotbarY + 2, 10, LIGHTGRAY);
                }
            }
        }

        EndDrawing();
    }

    for (int i = 1; i < 8; i++) {
        if (hasTex[i]) UnloadTexture(blockTex[i]);
    }
    CloseWindow();
    return 0;
}
