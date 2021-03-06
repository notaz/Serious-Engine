cmake_minimum_required(VERSION 2.8.7)
project(SeriousEngine)

# Set @rpath for Mac OS X shared library install names.
#cmake_policy(SET CMP0042 NEW)

# Use system SDL2 is on by default
option(USE_SYSTEM_SDL2 "Use system wide sdl2 libraries/includes" On)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")

# Use systemwide SDL2 or custom build
# RAKE!: Find a way to use their custom built library if
# they want to use that instead or if their system only
# allows for a setup like this.
if(NOT USE_SYSTEM_SDL2)
    include_directories(${CMAKE_SOURCE_DIR}/External/SDL2)
else()
    find_package(SDL2 REQUIRED)
    if(SDL2_FOUND)
	include_directories(${SDL2_INCLUDE_DIR})
    else()
	message(FATAL_ERROR "Error USE_SYSTEM_SDL2 is set but neccessary developer files not found")
    endif()
endif()

# Set up some sanity stuff...
if(CMAKE_SYSTEM_NAME STREQUAL "Linux" OR CMAKE_SYSTEM_NAME MATCHES "GNU|kFreeBSD")
    SET(LINUX TRUE)
endif()
if(CMAKE_SYSTEM_NAME STREQUAL "FreeBSD")
    SET(FREEBSD TRUE)
endif()
if(APPLE)
    SET(MACOSX TRUE)
endif()
if(MSVC)
    SET(WINDOWS TRUE)
endif()

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Debug CACHE STRING "None Debug Release RelWithDebInfo MinSizeRel" FORCE)
endif()
SET(DEBUG FALSE)
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    SET(DEBUG TRUE)
endif()

add_definitions(-D_REENTRANT=1)
add_definitions(-D_MT=1)

if(DEBUG)
    add_definitions(-DDEBUG=1)
    add_definitions(-D_DEBUG=1)
    set(DEBUGSUFFIX "D")
else()
    add_definitions(-DNDEBUG=1)
    add_definitions(-D_NDEBUG=1)
    set(DEBUGSUFFIX "")
endif()

if(WINDOWS)
    add_definitions(-DPLATFORM_WIN32=1)
    add_definitions(-DPRAGMA_ONCE=1)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS=1)
    add_definitions(-D_CRT_SECURE_NO_DEPRECATE=1)
endif()

if(LINUX)
    set(CMAKE_SKIP_RPATH ON CACHE BOOL "Skip RPATH" FORCE)
    add_definitions(-DPLATFORM_UNIX=1)
    add_definitions(-D_FILE_OFFSET_BITS=64)
    add_definitions(-D_LARGEFILE_SOURCE=1)
    add_definitions(-DPRAGMA_ONCE=1)
endif()

if(FREEBSD)
    set(CMAKE_SKIP_RPATH ON CACHE BOOL "Skip RPATH" FORCE)
    add_definitions(-DPLATFORM_UNIX=1)
    add_definitions(-DPLATFORM_FREEBSD=1)
    add_definitions(-D_FILE_OFFSET_BITS=64)
    add_definitions(-D_LARGEFILE_SOURCE=1)
    add_definitions(-DPRAGMA_ONCE=1)
    include_directories("/usr/local/include")
endif()

if(MACOSX)
    add_definitions(-DPLATFORM_UNIX=1)
    add_definitions(-DPLATFORM_MACOSX=1)
    add_definitions(-DPRAGMA_ONCE=1)
endif()

if(LINUX OR MACOSX)
    if(DEBUG)
        add_definitions(-O0 -ggdb3)
    endif(DEBUG)
    add_definitions(-pthread)
    add_definitions(-pipe -fsigned-char)
    #CXXFLAGS += -fexceptions -frtti
endif()

if(CMAKE_COMPILER_IS_GNUCC)
    # !!! FIXME: turn a bunch of these off, this is just for now. I hope.  --ryan.
    add_definitions(-Wno-invalid-offsetof)
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    # !!! FIXME: turn a bunch of these off, this is just for now. I hope.  --ryan.
    add_definitions(-Wno-unused-value)
    add_definitions(-Wno-switch)
    add_definitions(-Wno-tautological-undefined-compare)
    add_definitions(-Wno-c++11-compat-deprecated-writable-strings)
endif()

# !!! FIXME: you currently need this, but I'd like to flip this to not use
# !!! FIXME:  assembly language. And maybe delete the asm code too.
option(USE_I386_ASM "Use X86 ASM" TRUE)

if (USE_I386_ASM)
    # You need the Netwide Assembler (NASM) to build this on Intel systems.
    #   http://nasm.sf.net/
    add_definitions(-DUSE_I386_ASM=1)
    if (MACOSX)
        set(ASMOBJFMT "macho")
        list(APPEND ASMFLAGS --prefix _)
    elseif(WINDOWS)
        set(ASMOBJFMT "win32")
    else()
        set(ASMOBJFMT "elf")
    endif()
else()
    add_definitions(-DUSE_PORTABLE_C=1)
endif()

option(PANDORA "Compile for Pandora" FALSE)
if (PANDORA)
    add_definitions(-DPLATFORM_PANDORA=1)
endif()

option(USE_TREMOR "Use Tremor instead of Vorbis" FALSE)
if (USE_TREMOR)
    add_definitions(-DUSE_TREMOR=1)
endif()

option(TFE "Compile a The First Encounter version" FALSE)
if (TFE)
 add_definitions(-DFIRST_ENCOUNTER=1)
 set(MP "")
else()
 set(MP "MP")
endif()


# !!! FIXME: I currently force this, but you shouldn't _have_ to.
option(USE_SINGLE_THREAD "Use Single Threaded version" TRUE)
if(USE_SINGLE_THREAD)
 add_definitions(-DSINGLE_THREADED=1)
endif()


include_directories(
    .
    ${CMAKE_SOURCE_DIR}/External/libogg/include
)
if(USE_TREMOR)
if(PANDORA)
    include_directories(/mnt/utmp/codeblocks/usr/include/tremor)
else()
    # !!!Do something here!
endif()
else()
    include_directories(External/libvorbis/include)
endif()

# We build ECC, then use it to generate C++ code for the game entities...
macro(add_parser_and_scanner _PARSER _SCANNER)
    add_custom_command(
        OUTPUT "${CMAKE_CURRENT_SOURCE_DIR}/${_SCANNER}.cpp"
        MAIN_DEPENDENCY "${CMAKE_CURRENT_SOURCE_DIR}/${_SCANNER}.l"
        WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
        COMMAND flex
        ARGS -o${_SCANNER}.cpp ${_SCANNER}.l
    )

    add_custom_command(
        OUTPUT "${CMAKE_CURRENT_SOURCE_DIR}/${_PARSER}.cpp" "${CMAKE_CURRENT_SOURCE_DIR}/${_PARSER}.hpp"
        MAIN_DEPENDENCY "${CMAKE_CURRENT_SOURCE_DIR}/${_PARSER}.y"
        WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
        COMMAND bison
        ARGS -o${_PARSER}.cpp ${_PARSER}.y -d
    )

    add_custom_command(
        OUTPUT "${CMAKE_CURRENT_SOURCE_DIR}/${_PARSER}.h"
        MAIN_DEPENDENCY "${CMAKE_CURRENT_SOURCE_DIR}/${_PARSER}.hpp"
        WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
        COMMAND ${CMAKE_COMMAND}
        ARGS -E copy ${_PARSER}.hpp ${_PARSER}.h
    )
endmacro()

# Build ECC from source if there wasn't a prebuilt-one specified on the command line.
#  Normally we build it here, but we might need a prebuilt, native binary if
#  we're cross-compiling the rest of the game.
if(NOT ECC)
    add_parser_and_scanner("Ecc/Parser" "Ecc/Scanner")
    add_executable(ecc Ecc/Main.cpp Ecc/Parser.cpp Ecc/Parser.h Ecc/Scanner.cpp)
    set(ECC "ecc")
endif()

macro(entity _NAME)
    add_custom_command(
        OUTPUT "${CMAKE_CURRENT_SOURCE_DIR}/${_NAME}.cpp" "${CMAKE_CURRENT_SOURCE_DIR}/${_NAME}.h" "${CMAKE_CURRENT_SOURCE_DIR}/${_NAME}_tables.h"
        MAIN_DEPENDENCY "${CMAKE_CURRENT_SOURCE_DIR}/${_NAME}.es"
        DEPENDS ${ECC}
        WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
        COMMAND ${ECC}
        ARGS "${_NAME}.es"
    )
    list(APPEND ENTITIES_CPP "${CMAKE_CURRENT_SOURCE_DIR}/${_NAME}.cpp")
    list(APPEND ENTITIES_H "${CMAKE_CURRENT_SOURCE_DIR}/${_NAME}.h")
    list(APPEND ENTITIES_H "${CMAKE_CURRENT_SOURCE_DIR}/${_NAME}_tables.h")
endmacro()

set(ENTITIES_CPP "")
set(ENTITIES_H "")
entity(Engine/Classes/BaseEvents)
entity(Engine/Classes/MovableBrushEntity)
entity(Engine/Classes/MovableEntity)
entity(Engine/Classes/MovableModelEntity)
entity(Engine/Classes/PlayerEntity)
set(ENGINE_ENTITIES_CPP ${ENTITIES_CPP})

set(ENTITIES_CPP "")
if(TFE)
 entity(Entities/Acid)
 entity(Entities/AirWave)
 entity(Entities/AmmoItem)
 entity(Entities/AmmoPack)
 entity(Entities/AnimationChanger)
 entity(Entities/AnimationHub)
 entity(Entities/ArmorItem)
 entity(Entities/BackgroundViewer)
 entity(Entities/BasicEffects)
 entity(Entities/Beast)
 entity(Entities/BigHead)
 entity(Entities/BlendController)
 entity(Entities/BloodSpray)
 entity(Entities/Boneman)
 entity(Entities/Bouncer)
 entity(Entities/Bullet)
 entity(Entities/Camera)
 entity(Entities/CameraMarker)
 entity(Entities/CannonBall)
 entity(Entities/Catman)
 entity(Entities/Copier)
 entity(Entities/Counter)
 entity(Entities/CrateRider)
 entity(Entities/CyborgBike)
 entity(Entities/Cyborg)
 entity(Entities/Damager)
 entity(Entities/Debris)
 entity(Entities/DestroyableArchitecture)
 entity(Entities/Devil)
 entity(Entities/DevilMarker)
 entity(Entities/DevilProjectile)
 entity(Entities/DoorController)
 entity(Entities/Dragonman)
 entity(Entities/EffectMarker)
 entity(Entities/Effector)
 entity(Entities/Elemental)
 entity(Entities/EnemyBase)
 entity(Entities/EnemyCounter)
 entity(Entities/EnemyDive)
 entity(Entities/EnemyFly)
 entity(Entities/EnemyMarker)
 entity(Entities/EnemyRunInto)
 entity(Entities/EnemySpawner)
 entity(Entities/EnvironmentBase)
 entity(Entities/EnvironmentMarker)
 entity(Entities/Eruptor)
 entity(Entities/Eyeman)
 entity(Entities/Fish)
 entity(Entities/Fishman)
 entity(Entities/Flame)
 entity(Entities/FogMarker)
 #entity(Entities/GhostBusterRay)
 entity(Entities/Gizmo)
 entity(Entities/Global)
 entity(Entities/GradientMarker)
 entity(Entities/GravityMarker)
 entity(Entities/GravityRouter)
 entity(Entities/HazeMarker)
 entity(Entities/Headman)
 entity(Entities/HealthItem)
 entity(Entities/Huanman)
 entity(Entities/Item)
 entity(Entities/KeyItem)
 entity(Entities/Light)
 entity(Entities/Lightning)
 entity(Entities/LightStyle)
 entity(Entities/Mamut)
 entity(Entities/Mamutman)
 entity(Entities/Mantaman)
 entity(Entities/Marker)
 entity(Entities/MessageHolder)
 entity(Entities/MessageItem)
 entity(Entities/MirrorMarker)
 entity(Entities/ModelDestruction)
 entity(Entities/ModelHolder2)
 entity(Entities/ModelHolder)
 entity(Entities/MovingBrush)
 entity(Entities/MovingBrushMarker)
 entity(Entities/MusicChanger)
 entity(Entities/MusicHolder)
 entity(Entities/NavigationMarker)
 entity(Entities/ParticlesHolder)
 entity(Entities/Pendulum)
 entity(Entities/Pipebomb)
 entity(Entities/PlayerActionMarker)
 entity(Entities/PlayerAnimator)
 entity(Entities/Player)
 entity(Entities/PlayerMarker)
 entity(Entities/PlayerView)
 entity(Entities/PlayerWeaponsEffects)
 entity(Entities/PlayerWeapons)
 entity(Entities/Projectile)
 entity(Entities/PyramidSpaceShip)
 entity(Entities/PyramidSpaceShipMarker)
 entity(Entities/Reminder)
 entity(Entities/RobotDriving)
 entity(Entities/RobotFixed)
 entity(Entities/RobotFlying)
 entity(Entities/RollingStone)
 entity(Entities/Scorpman)
 entity(Entities/Ship)
 entity(Entities/ShipMarker)
 entity(Entities/SoundHolder)
 entity(Entities/StormController)
 entity(Entities/Switch)
 entity(Entities/Teleport)
 entity(Entities/TouchField)
 entity(Entities/Trigger)
 entity(Entities/Twister)
 entity(Entities/VoiceHolder)
 entity(Entities/Walker)
 entity(Entities/Watcher)
 entity(Entities/WatchPlayers)
 entity(Entities/Water)
 entity(Entities/WeaponItem)
 entity(Entities/Werebull)
 entity(Entities/Woman)
 entity(Entities/WorldBase)
 entity(Entities/WorldLink)
 entity(Entities/WorldSettingsController)
else()
 entity(EntitiesMP/AirElemental)
 entity(EntitiesMP/AirShockwave)
 entity(EntitiesMP/AmmoItem)
 entity(EntitiesMP/AmmoPack)
 entity(EntitiesMP/AnimationChanger)
 entity(EntitiesMP/AnimationHub)
 entity(EntitiesMP/AreaMarker)
 entity(EntitiesMP/ArmorItem)
 entity(EntitiesMP/BackgroundViewer)
 entity(EntitiesMP/BasicEffects)
 entity(EntitiesMP/Beast)
 entity(EntitiesMP/BigHead)
 entity(EntitiesMP/BlendController)
 entity(EntitiesMP/BloodSpray)
 entity(EntitiesMP/Boneman)
 entity(EntitiesMP/Bouncer)
 entity(EntitiesMP/Bullet)
 entity(EntitiesMP/Camera)
 entity(EntitiesMP/CameraMarker)
 entity(EntitiesMP/CannonBall)
 entity(EntitiesMP/CannonRotating)
 entity(EntitiesMP/CannonStatic)
 entity(EntitiesMP/ChainsawFreak)
 entity(EntitiesMP/Copier)
 entity(EntitiesMP/Counter)
 entity(EntitiesMP/CrateBus)
 entity(EntitiesMP/CrateRider)
 entity(EntitiesMP/CreditsHolder)
 entity(EntitiesMP/Damager)
 entity(EntitiesMP/Debris)
 entity(EntitiesMP/DebugEntityStatesDisplay)
 entity(EntitiesMP/Demon)
 entity(EntitiesMP/DestroyableArchitecture)
 entity(EntitiesMP/Devil)
 entity(EntitiesMP/DevilMarker)
 entity(EntitiesMP/DevilProjectile)
 entity(EntitiesMP/DoorController)
 entity(EntitiesMP/Dragonman)
 entity(EntitiesMP/EffectMarker)
 entity(EntitiesMP/Effector)
 entity(EntitiesMP/Elemental)
 entity(EntitiesMP/EnemyBase)
 entity(EntitiesMP/EnemyCounter)
 entity(EntitiesMP/EnemyDive)
 entity(EntitiesMP/EnemyFly)
 entity(EntitiesMP/EnemyMarker)
 entity(EntitiesMP/EnemyRunInto)
 entity(EntitiesMP/EnemySpawner)
 entity(EntitiesMP/EnvironmentBase)
 entity(EntitiesMP/EnvironmentMarker)
 entity(EntitiesMP/EnvironmentParticlesHolder)
 entity(EntitiesMP/Eruptor)
 entity(EntitiesMP/ExotechLarva)
 entity(EntitiesMP/ExotechLarvaBattery)
 entity(EntitiesMP/ExotechLarvaCharger)
 entity(EntitiesMP/Eyeman)
 entity(EntitiesMP/Fireworks)
 entity(EntitiesMP/Fish)
 entity(EntitiesMP/Flame)
 entity(EntitiesMP/FogMarker)
 entity(EntitiesMP/Gizmo)
 entity(EntitiesMP/Global)
 entity(EntitiesMP/GradientMarker)
 entity(EntitiesMP/GravityMarker)
 entity(EntitiesMP/GravityRouter)
 entity(EntitiesMP/Grunt)
 entity(EntitiesMP/GruntSka)
 entity(EntitiesMP/Guffy)
 entity(EntitiesMP/HazeMarker)
 entity(EntitiesMP/Headman)
 entity(EntitiesMP/HealthItem)
 entity(EntitiesMP/HudPicHolder)
 entity(EntitiesMP/Item)
 entity(EntitiesMP/KeyItem)
 entity(EntitiesMP/LarvaOffspring)
 entity(EntitiesMP/Light)
 entity(EntitiesMP/Lightning)
 entity(EntitiesMP/Marker)
 entity(EntitiesMP/MessageHolder)
 entity(EntitiesMP/MessageItem)
 entity(EntitiesMP/MeteorShower)
 entity(EntitiesMP/MirrorMarker)
 entity(EntitiesMP/ModelDestruction)
 entity(EntitiesMP/ModelHolder)
 entity(EntitiesMP/ModelHolder2)
 entity(EntitiesMP/ModelHolder3)
 entity(EntitiesMP/MovingBrush)
 entity(EntitiesMP/MovingBrushMarker)
 entity(EntitiesMP/MusicChanger)
 entity(EntitiesMP/MusicHolder)
 entity(EntitiesMP/NavigationMarker)
 entity(EntitiesMP/ParticlesHolder)
 entity(EntitiesMP/Pendulum)
 entity(EntitiesMP/PhotoAlbum)
 entity(EntitiesMP/Pipebomb)
 entity(EntitiesMP/Player)
 entity(EntitiesMP/PlayerActionMarker)
 entity(EntitiesMP/PlayerAnimator)
 entity(EntitiesMP/PlayerMarker)
 entity(EntitiesMP/PlayerView)
 entity(EntitiesMP/PlayerWeapons)
 entity(EntitiesMP/PlayerWeaponsEffects)
 entity(EntitiesMP/PowerUpItem)
 entity(EntitiesMP/Projectile)
 entity(EntitiesMP/PyramidSpaceShip)
 entity(EntitiesMP/PyramidSpaceShipMarker)
 entity(EntitiesMP/Reminder)
 entity(EntitiesMP/RollingStone)
 entity(EntitiesMP/Santa)
 entity(EntitiesMP/Scorpman)
 entity(EntitiesMP/ScrollHolder)
 entity(EntitiesMP/SeriousBomb)
 entity(EntitiesMP/Ship)
 entity(EntitiesMP/ShipMarker)
 entity(EntitiesMP/Shooter)
 entity(EntitiesMP/SoundHolder)
 entity(EntitiesMP/SpawnerProjectile)
 entity(EntitiesMP/Spinner)
 entity(EntitiesMP/StormController)
 entity(EntitiesMP/Summoner)
 entity(EntitiesMP/SummonerMarker)
 entity(EntitiesMP/Switch)
 entity(EntitiesMP/TacticsChanger)
 entity(EntitiesMP/TacticsHolder)
 entity(EntitiesMP/Teleport)
 entity(EntitiesMP/Terrain)
 entity(EntitiesMP/TextFXHolder)
 entity(EntitiesMP/TimeController)
 entity(EntitiesMP/TouchField)
 entity(EntitiesMP/Trigger)
 entity(EntitiesMP/Twister)
 entity(EntitiesMP/VoiceHolder)
 entity(EntitiesMP/Walker)
 entity(EntitiesMP/WatchPlayers)
 entity(EntitiesMP/Watcher)
 entity(EntitiesMP/Water)
 entity(EntitiesMP/WeaponItem)
 entity(EntitiesMP/Werebull)
 entity(EntitiesMP/Woman)
 entity(EntitiesMP/WorldBase)
 entity(EntitiesMP/WorldLink)
 entity(EntitiesMP/WorldSettingsController)
endif()

add_custom_target(ParseEntities DEPENDS ${ENTITIES_H})

set(ENTITIESMPLIB "Entities${MP}${DEBUGSUFFIX}")
if(TFE)
 add_library(${ENTITIESMPLIB} SHARED
     ${ENTITIES_CPP}
     Entities/Common/Common.cpp
     Entities/Common/Debris.cpp
     Entities/Common/Particles.cpp
     Entities/Common/Stats.cpp
     Entities/Common/PathFinding.cpp
     Entities/Common/HUD.cpp
 )
else()
 add_library(${ENTITIESMPLIB} SHARED
     ${ENTITIES_CPP}
     EntitiesMP/Common/Common.cpp
 	 EntitiesMP/Common/Particles.cpp
 	 EntitiesMP/Common/EmanatingParticles.cpp
 	 EntitiesMP/Common/PathFinding.cpp
 	 EntitiesMP/Common/HUD.cpp
 )
endif()

if(MACOSX)
    target_link_libraries(${ENTITIESMPLIB} "-undefined dynamic_lookup")
endif()
add_dependencies(${ENTITIESMPLIB} ParseEntities)

set(GAMEMPLIB "Game${MP}${DEBUGSUFFIX}")
add_library(${GAMEMPLIB} SHARED
    GameMP/Camera.cpp
    GameMP/CompMessage.cpp
    GameMP/CompModels.cpp
    GameMP/Computer.cpp
    GameMP/Console.cpp
    GameMP/Controls.cpp
    GameMP/Game.cpp
    GameMP/LCDDrawing.cpp
    GameMP/LoadingHook.cpp
    GameMP/Map.cpp
    GameMP/SessionProperties.cpp
    GameMP/WEDInterface.cpp
)
if(MACOSX)
    target_link_libraries(${GAMEMPLIB} "-undefined dynamic_lookup")
endif()
add_dependencies(${GAMEMPLIB} ParseEntities)

set(SHADERSLIB "Shaders${DEBUGSUFFIX}")
add_library(${SHADERSLIB} SHARED
    Shaders/AddShader.cpp
    Shaders/AddShaderDS.cpp
    Shaders/BaseShader.cpp
    Shaders/BaseShaderDS.cpp
    Shaders/BaseTransparent.cpp
    Shaders/BaseTransparentDS.cpp
    Shaders/ColorShader.cpp
    Shaders/Common.cpp
    Shaders/DetailShader.cpp
    Shaders/DisplaceShader.cpp
    Shaders/InvisibleShader.cpp
    Shaders/MultiLayerShader.cpp
    Shaders/Reflection.cpp
    Shaders/ReflectionDS.cpp
    Shaders/ReftectionAndSpecular.cpp
    Shaders/ReftectionAndSpecularDS.cpp
    Shaders/Specular.cpp
    Shaders/SpecularDS.cpp
    Shaders/StdH.cpp
    Shaders/Translucent.cpp
)
if(MACOSX)
    target_link_libraries(${SHADERSLIB} "-undefined dynamic_lookup")
endif()
add_dependencies(${SHADERSLIB} ParseEntities)

add_parser_and_scanner("Engine/Base/Parser" "Engine/Base/Scanner")
add_parser_and_scanner("Engine/Ska/smcPars" "Engine/Ska/smcScan")

if (USE_I386_ASM)
    add_custom_command(
        OUTPUT "SoundMixer386.o"
        MAIN_DEPENDENCY "${CMAKE_CURRENT_SOURCE_DIR}/Engine/Sound/SoundMixer386.asm"
        COMMAND nasm
        ARGS ${ASMFLAGS} -f ${ASMOBJFMT} -o SoundMixer386.o ${CMAKE_CURRENT_SOURCE_DIR}/Engine/Sound/SoundMixer386.asm
    )
    list(APPEND ADDITIONAL_ENGINE_SRCS SoundMixer386.o)
endif()

if(USE_SINGLE_THREAD)
 set(SYNCHRO_SRCS
    Engine/Base/NullSynchronization.cpp  # single threaded.
 )
else()
 #!! TODO Win32/Linux case
 set(SYNCHRO_SRCS
    Engine/Base/Unix/UnixSynchronization.cpp  # multithreaded Unix.
    #Engine/Base/SDL/SDLSynchronization.cpp
    #Engine/Base/SDL/SDLThreadLocalStorage.cpp  # multithreaded Unix.
 )
    #Engine/Base/Registry.cpp  # Windows only.
    #Engine/Base/StackDump.cpp  # Windows only.
    #Engine/Base/Win32/Win32Synchronization.cpp # Windows only.
endif()

set(ENGINE_SRCS
    ${ENGINE_ENTITIES_CPP}
    Engine/Engine.cpp
    Engine/Base/Anim.cpp
    Engine/Base/CRC.cpp
    Engine/Base/CRCTable.cpp
    Engine/Base/Changeable.cpp
    Engine/Base/Console.cpp
    Engine/Base/Directory.cpp
    Engine/Base/ErrorReporting.cpp
    Engine/Base/FileName.cpp
    Engine/Base/Input.cpp
    Engine/Base/Lists.cpp
    Engine/Base/Memory.cpp
    Engine/Base/Profiling.cpp
    Engine/Base/ProgressHook.cpp
    Engine/Base/Protection.cpp
    Engine/Base/Relations.cpp
    Engine/Base/ReplaceFile.cpp
    Engine/Base/Serial.cpp
    Engine/Base/Shell.cpp
    Engine/Base/ShellTypes.cpp
    Engine/Base/Statistics.cpp
    Engine/Base/Stream.cpp
    Engine/Base/Timer.cpp
    Engine/Base/Translation.cpp
    Engine/Base/Unzip.cpp
    Engine/Base/Updateable.cpp
    Engine/Base/CTString.cpp
    Engine/Base/Scanner.cpp
    Engine/Base/Parser.cpp
    Engine/Base/Parser.h
    Engine/Base/IFeel.cpp
    Engine/Base/Unix/UnixFileSystem.cpp
    Engine/Base/Unix/UnixDynamicLoader.cpp
    Engine/Base/SDL/SDLTimer.cpp
    Engine/Base/SDL/SDLInput.cpp
    Engine/Base/SDL/SDLEvents.cpp
    ${SYNCHRO_SRCS}
    Engine/Brushes/Brush.cpp
    Engine/Brushes/BrushIO.cpp
    Engine/Brushes/BrushShadows.cpp
    Engine/Brushes/BrushTriangularize.cpp
    Engine/Brushes/BrushArchive.cpp
    Engine/Brushes/BrushImport.cpp
    Engine/Brushes/BrushMip.cpp
    Engine/Brushes/BrushPolygon.cpp
    Engine/Brushes/BrushExport.cpp
    Engine/Brushes/BrushSector.cpp
    Engine/Entities/Entity.cpp
    Engine/Entities/NearestPolygon.cpp
    Engine/Entities/EntityProperties.cpp
    Engine/Entities/PlayerCharacter.cpp
    Engine/Entities/EntityClass.cpp
    Engine/Entities/FieldBSPTesting.cpp
    Engine/Entities/EntityCollision.cpp
    Engine/Entities/EntityCopying.cpp
    Engine/Entities/LastPositions.cpp
    Engine/Math/Projection_Isometric.cpp
    Engine/Math/Object3D.cpp
    Engine/Math/Projection_Parallel.cpp
    Engine/Math/Projection_Perspective.cpp
    Engine/Math/Float.cpp
    Engine/Math/Object3D_CSG.cpp
    Engine/Math/Projection_Simple.cpp
    Engine/Math/Projection_Simple_DOUBLE.cpp
    Engine/Math/Functions.cpp
    Engine/Math/ObjectSector.cpp
    Engine/Math/Placement.cpp
    Engine/Math/TextureMapping.cpp
    Engine/Math/Geometry.cpp
    Engine/Math/Projection.cpp
    Engine/Math/Geometry_DOUBLE.cpp
    #Engine/Math/Object3D_IO.cpp  # Exploration 3D support.
    #Engine/Models/EditModel.cpp
    Engine/Models/Model.cpp
    Engine/Models/RenderModel_View.cpp
    Engine/Models/Normals.cpp
    Engine/Models/VertexGetting.cpp
    Engine/Models/RenderModel.cpp
    Engine/Models/MipMaker.cpp
    Engine/Models/ModelProfile.cpp
    Engine/Models/RenderModel_Mask.cpp
    Engine/Light/LayerMaker.cpp
    Engine/Light/LayerMixer.cpp
    Engine/Light/LightSource.cpp
    Engine/Graphics/Adapter.cpp
    Engine/Graphics/Raster.cpp
    Engine/Graphics/GfxLibrary.cpp
    Engine/Graphics/Benchmark.cpp
    Engine/Graphics/GfxProfile.cpp
    Engine/Graphics/Color.cpp
    Engine/Graphics/ShadowMap.cpp
    Engine/Graphics/DepthCheck.cpp
    Engine/Graphics/Texture.cpp
    Engine/Graphics/DisplayMode.cpp
    Engine/Graphics/Gfx_OpenGL.cpp
    Engine/Graphics/Gfx_OpenGL_Textures.cpp
    Engine/Graphics/TextureEffects.cpp
    Engine/Graphics/DrawPort.cpp
    Engine/Graphics/Gfx_wrapper.cpp
    Engine/Graphics/DrawPort_Particles.cpp
    Engine/Graphics/Graphics.cpp
    Engine/Graphics/ViewPort.cpp
    Engine/Graphics/DrawPort_RenderScene.cpp
    Engine/Graphics/ImageInfo.cpp
    Engine/Graphics/Fog.cpp
    Engine/Graphics/MultiMonitor.cpp
    Engine/Graphics/Font.cpp
    Engine/Graphics/Shader.cpp
    Engine/Graphics/Stereo.cpp
    Engine/Graphics/SDL/SDLOpenGL.cpp
    Engine/Graphics/SDL/SDLAdapter.cpp
    Engine/Network/ActionBuffer.cpp
    Engine/Network/NetworkMessage.cpp
    Engine/Network/Server.cpp
    Engine/Network/Buffer.cpp
    Engine/Network/NetworkProfile.cpp
    Engine/Network/SessionState.cpp
    Engine/Network/PlayerBuffer.cpp
    Engine/Network/MessageDispatcher.cpp
    Engine/Network/PlayerSource.cpp
    Engine/Network/Compression.cpp
    Engine/Network/Network.cpp
    Engine/Network/PlayerTarget.cpp
    Engine/Network/CPacket.cpp
    Engine/Network/ClientInterface.cpp
    Engine/Network/CommunicationInterface.cpp
    Engine/Network/Diff.cpp
    Engine/GameAgent/GameAgent.cpp
    Engine/Terrain/ArrayHolder.cpp
    Engine/Terrain/Terrain.cpp
    Engine/Terrain/TerrainArchive.cpp
    Engine/Terrain/TerrainEditing.cpp
    Engine/Terrain/TerrainLayer.cpp
    Engine/Terrain/TerrainMisc.cpp
    Engine/Terrain/TerrainRayCasting.cpp
    Engine/Terrain/TerrainRender.cpp
    Engine/Terrain/TerrainTile.cpp
    Engine/Rendering/Render.cpp
    Engine/Rendering/RenderProfile.cpp
    Engine/Rendering/SelectOnRender.cpp
    Engine/Ska/AnimSet.cpp
    Engine/Ska/RMRender.cpp
    Engine/Ska/Skeleton.cpp
    Engine/Ska/ModelInstance.cpp
    Engine/Ska/StringTable.cpp
    Engine/Ska/Mesh.cpp
    Engine/Ska/RMRenderMask.cpp
    Engine/Ska/smcPars.cpp
    Engine/Ska/smcPars.h
    Engine/Ska/smcScan.cpp
    Engine/Sound/SoundDecoder.cpp
    Engine/Sound/SoundObject.cpp
    Engine/Sound/SoundLibrary.cpp
    Engine/Sound/SoundProfile.cpp
    Engine/Sound/SoundData.cpp
    Engine/Sound/Wave.cpp
    Engine/Sound/SoundMixer.cpp
    Engine/Templates/Stock_CAnimData.cpp
    Engine/Templates/Stock_CAnimSet.cpp
    Engine/Templates/Stock_CEntityClass.cpp
    Engine/Templates/Stock_CMesh.cpp
    Engine/Templates/Stock_CModelData.cpp
    Engine/Templates/Stock_CSkeleton.cpp
    Engine/Templates/Stock_CSoundData.cpp
    Engine/Templates/Stock_CTextureData.cpp
    Engine/Templates/Stock_CShader.cpp
    Engine/Templates/NameTable_CTFileName.cpp
    Engine/Templates/NameTable_CTranslationPair.cpp
    Engine/Templates/BSP.cpp
    Engine/World/WorldCSG.cpp
    Engine/World/PhysicsProfile.cpp
    Engine/World/WorldCollision.cpp
    Engine/World/WorldIO.cpp
    Engine/World/WorldRayCasting.cpp
    Engine/World/World.cpp
    Engine/World/WorldCollisionGrid.cpp
    Engine/World/WorldEditingProfile.cpp
    Engine/zlib/inflate.c
    Engine/zlib/adler32.c
    Engine/zlib/infblock.c
    Engine/zlib/inffast.c
    Engine/zlib/inftrees.c
    Engine/zlib/trees.c
    Engine/zlib/compress.c
    Engine/zlib/zutil.c
    Engine/zlib/deflate.c
    Engine/zlib/infcodes.c
    Engine/zlib/infutil.c
    Engine/zlib/uncompr.c
    ${ADDITIONAL_ENGINE_SRCS}
)

add_executable(ssam
    ${ENGINE_SRCS}
    SeriousSam/LevelInfo.cpp
    SeriousSam/CmdLine.cpp
    SeriousSam/SeriousSam.cpp
    SeriousSam/VarList.cpp
    SeriousSam/Credits.cpp
    SeriousSam/GLSettings.cpp
    SeriousSam/LCDDrawing.cpp
    SeriousSam/SplashScreen.cpp
    SeriousSam/MainWindow.cpp
    SeriousSam/Menu.cpp
    SeriousSam/MenuGadgets.cpp
    SeriousSam/MenuPrinting.cpp
)
add_dependencies(ssam ParseEntities)
# Make symbols in the main executable available to dynamic objects
set_target_properties(ssam PROPERTIES ENABLE_EXPORTS ON)

# !!! FIXME: this is an option because you have to recompile the entire engine twice.
# !!! FIXME:  If we can put the engine in a static library and not lose symbols,
# !!! FIXME:  that's a better plan and we can remove the toggle here.
option(BUILD_DEDICATED_SERVER "Compile the dedicated server, too" FALSE)
if(BUILD_DEDICATED_SERVER)
    add_executable(SeriousSamDedicated ${ENGINE_SRCS} DedicatedServer/DedicatedServer.cpp)
    add_dependencies(SeriousSamDedicated ParseEntities)
    # Make symbols in the main executable available to dynamic objects
    set_target_properties(SeriousSamDedicated PROPERTIES ENABLE_EXPORTS ON)
endif()

if(MACOSX)
    find_library(COCOA_FRAMEWORK Cocoa)
    target_link_libraries(ssam "${COCOA_FRAMEWORK}")
    target_link_libraries(ssam "${CMAKE_CURRENT_SOURCE_DIR}/lib/macosx/libSDL2-2.0.0.dylib")
    if(BUILD_DEDICATED_SERVER)
        target_link_libraries(SeriousSamDedicated "${COCOA_FRAMEWORK}")
        target_link_libraries(SeriousSamDedicated "${CMAKE_CURRENT_SOURCE_DIR}/lib/macosx/libSDL2-2.0.0.dylib")
    endif()
endif()

if(LINUX)
    set_target_properties(ssam PROPERTIES LINK_FLAGS "-Wl,-rpath,$ORIGIN")
    target_link_libraries(ssam "m")
    target_link_libraries(ssam "dl")
    if(PANDORA)
    target_link_libraries(ssam "rt")
    target_link_libraries(ssam ${SDL2_LIBRARY}) # RAKE!: I don't know if this will work yet.
    endif()
    target_link_libraries(ssam "pthread")
    target_link_libraries(ssam ${SDL2_LIBRARY})
    if(BUILD_DEDICATED_SERVER)
        set_target_properties(SeriousSamDedicated PROPERTIES LINK_FLAGS "-Wl,-rpath,$ORIGIN")
        target_link_libraries(SeriousSamDedicated "m")
        target_link_libraries(SeriousSamDedicated "dl")
        target_link_libraries(SeriousSamDedicated "pthread")
	target_link_libraries(SeriousSamDedicated ${SDL2_LIBRARY})
    endif()
endif()

if(FREEBSD)
    set_target_properties(ssam PROPERTIES LINK_FLAGS "-Wl,-rpath,$ORIGIN")
    target_link_libraries(ssam "m")
    target_link_libraries(ssam "pthread")
    target_link_libraries(ssam ${SDL2_LIBRARY})
    if(BUILD_DEDICATED_SERVER)
        set_target_properties(SeriousSamDedicated PROPERTIES LINK_FLAGS "-Wl,-rpath,$ORIGIN")
        target_link_libraries(SeriousSamDedicated "m")
        target_link_libraries(SeriousSamDedicated "pthread")
        target_link_libraries(SeriousSamDedicated ${SDL2_LIBRARY})
    endif()
endif()

if(TFE)
 set_target_properties(ssam PROPERTIES OUTPUT_NAME "ssam-tfe")
endif()

# end of CMakeLists.txt ...

