# micropython-sunfish

MicroPython Sunfish Chess Engine

This is an unofficial MicroPython port of [Sunfish](https://github.com/thomasahle/sunfish/tree/83d675df2f51cd2c66cc765919da7425500a29b6) Chess Engine.
Some parameters (marked with QL) are tuned to make it fit constrained memory.
It is tested with MicroPython V1.19.1 on ESP32.

  [sunfish_github]: https://github.com/thomasahle/sunfish

- Ported by: Quan Lin
- License: GNU GPL v3 (the same as Sunfish)

# Installation

The file may be installed on target hardware with `mpremote`:
```bash
$ mpremote mip install github:/jacklinquan/micropython-sunfish/sunfish.py
```

# Gameplay

Running `sunfish.py` starts a terminal-based textual chess game. The human player is White,
and the machine is Black. White pieces are denoted by upper case letters, and Black by lower. Blank
squares are marked with a `.`character. The appearance of the board is as follows:
```
  8 r n b q k b n r
  7 p p p p . p p p
  6 . . . . . . . .
  5 . . . . p . . .
  4 . . . . P . . .
  3 . . . . . . . .
  2 P P P P . P P P
  1 R N B Q K B N R
    a b c d e f g h 
```
Moves are input and displayed in "algebraic" notation, for example `b1c3` to move a white Knight.
Castling is done by moving the King, e.g. `e1g1` to castle on the King side - the Rook moves
automatically. Pawn promotion - always to Queen - occurs automatically. The machine can do _en passant_
captures. I haven't had the opportunity to test a human _en passant_ move but I would expect it to be
performed by moving the Pawn to its end position.

If an invalid move is entered, the engine prompts the user to try again.

## Limitations

The machine does not tell you when it has put you in check. If you fail to deal with the situation it
will tell you that you have lost. It can also fail to notice that it is in checkmate. You need to
actually take its King for it to acknowledge the loss.

As is often the case for simple engines, its end-game performance is poor. In one game where I was
reduced to just my King, it failed to spot an excruciatingly obvious mate-in-one while it set about
promoting a Pawn to add a Queen to its two Rooks...

The engine does not allow you to play as Black.

# API

This allows Sunfish to work with a graphical front-end such as
[miropython-touch](https://github.com/peterhinch/micropython-touch/tree/master). It is intended that
the API could be adapted to other chess engines to enable engines and front-ends to be ported.

The API comprises two generators: `game` and `get_board`. With suitable front-end code the user can
play as Black or White.

## The game generator

This runs for the duration of a game. The end of a game is flagged by a `StopIteration` exception,
with the exception value indicating win, lose or (potentially) draw. (Sunfish cannot detect draws).
By default the generator starts with the Sunfish default board, the user playing White. If the user
enters an invalid move, it is ignored. Special moves such as castling are handled as described above
in the text game.

The following is an example of the generator in use

```py
    async def play_game(self):
        game_over = False
        # Set up the board
        game = sf.game(iblack) if self.invert else sf.game()
        pos, _ = next(game)  # Start generator, get initial position
        while not game_over:
            try:
                self.populate(pos)  # Fill the board grid with the current position
                await asyncio.sleep(0)
                pos = None
                while pos is None:  # Acquire valid move
                    await self.moved.wait()  # Wait for player/GUI to update self.move
                    pos = game.send(self.move)  # Get position after move
                self.populate(pos)  # Use the Position instance to redraw the board
                await asyncio.sleep(1)  # Ensure refresh, allow time to view.
                pos, mvblack = next(game)  # Sunfish calculates its move. Yields the new board state and its move
                self.flash(*rc(mvblack[:2]), WHITE)  # Flash the squares to make the forthcoming move obvious
                self.flash(*rc(mvblack[2:]), WHITE)
                await asyncio.sleep_ms(700)  # Let user see forthcoming move
            except StopIteration as e:
                game_over = True
                print(f"Game over: you {'won' if e else 'lost'}")
```
Note that the generator iterations return a `Position` instance. From the point of view of the GUI code
this is an opaque object whose only use is to be passed to the `get_board` generator. In general a
`Position` is some object which contains information on the current board state.



### Inverted play

It is possible to pass a board state to the generator function to enable an initial White move with
the user playing as Black. In this case the GUI front end shows lowercase pieces as White and uppercase
as Black, so the board appears with the user's Black pieces at the bottom.

The way of passing the initial state is a WIP.


## The get_board generator

This is instantiated with a `Position` object, as returned by the `game` generator, and yields the
alphabetic character associated with each square in turn. Order is left to right, then top to bottom.
In the board illustrated above, successive calls would yield "rnbqkbnrpppp.ppp." The following is a code
fragment illustrating its use to populate a grid with the current board state.
```py
# Return state of each square in turn.
def get(pos, invert):
    dic = {}
    gen = sunfish.get_board(pos)  # Instantiate Sunfish generator
    n = 0
    while True:
        r = next(gen)  # Get chesspiece character
        dic["text"] = lut[r.upper()]  # Convert to chess glyph
        dic["fgcolor"] = WHITE if (r.isupper() ^ invert) else RED
        dic["bgcolor"] = BLACK if ((n ^ (n >> 3)) & 1) else PALE_GREY
        yield dic
        n += 1
```
