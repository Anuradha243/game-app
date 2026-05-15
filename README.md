# game-app
application of game
<img width="1581" height="779" alt="Screenshot 2026-05-14 165312" src="https://github.com/user-attachments/assets/1f49bfa1-f197-4507-8729-7f46e3f0f1f8" />










import { useState, useCallback } from "react";

const COLORS = ["red", "blue", "green", "yellow"];
const COLOR_HEX = { red: "#e74c3c", blue: "#3498db", green: "#2ecc71", yellow: "#f1c40f" };
const COLOR_DARK = { red: "#c0392b", blue: "#2980b9", green: "#27ae60", yellow: "#f39c12" };
const COLOR_LIGHT = { red: "#fadbd8", blue: "#d6eaf8", green: "#d5f5e3", yellow: "#fef9e7" };

// Home positions for each color (row, col) on 15x15 grid
const HOME_COLS = { red: [1,1], blue: [1,9], green: [9,9], yellow: [9,1] };
const PIECE_HOME = {
  red: [[1,1],[1,2],[2,1],[2,2]],
  blue: [[1,9],[1,10],[2,9],[2,10]],
  green: [[9,9],[9,10],[10,9],[10,10]],
  yellow: [[9,1],[9,2],[10,1],[10,2]],
};

// Main path (row, col) - 52 squares
const PATH = [
  [6,1],[6,2],[6,3],[6,4],[6,5],
  [5,6],[4,6],[3,6],[2,6],[1,6],[0,6],
  [0,7],
  [0,8],[1,8],[2,8],[3,8],[4,8],[5,8],
  [6,9],[6,10],[6,11],[6,12],[6,13],[6,14],
  [7,14],
  [8,14],[8,13],[8,12],[8,11],[8,10],[8,9],
  [9,8],[10,8],[11,8],[12,8],[13,8],[14,8],
  [14,7],
  [14,6],[13,6],[12,6],[11,6],[10,6],[9,6],
  [8,5],[8,4],[8,3],[8,2],[8,1],[8,0],
  [7,0],
];

// Start index on PATH for each color
const START_IDX = { red: 0, blue: 13, green: 26, yellow: 39 };

// Home column path (the colored stretch before center)
const HOME_PATH = {
  red: [[7,1],[7,2],[7,3],[7,4],[7,5],[7,6]],
  blue: [[1,7],[2,7],[3,7],[4,7],[5,7],[6,7]],
  green: [[7,13],[7,12],[7,11],[7,10],[7,9],[7,8]],
  yellow: [[13,7],[12,7],[11,7],[10,7],[9,7],[8,7]],
};

const CENTER = [7,7];

function posKey(r,c){ return `${r},${c}`; }

function initPieces() {
  const pieces = {};
  COLORS.forEach(color => {
    pieces[color] = PIECE_HOME[color].map((pos, i) => ({
      id: i, color, state: "home", pathIdx: -1, homeIdx: -1,
      row: pos[0], col: pos[1]
    }));
  });
  return pieces;
}

export default function Ludo() {
  const [pieces, setPieces] = useState(initPieces());
  const [turn, setTurn] = useState(0); // index into COLORS
  const [dice, setDice] = useState(null);
  const [rolled, setRolled] = useState(false);
  const [selected, setSelected] = useState(null);
  const [msg, setMsg] = useState("Red's turn — Roll the dice!");
  const [winner, setWinner] = useState(null);
  const [rolling, setRolling] = useState(false);

  const currentColor = COLORS[turn];

  const rollDice = () => {
    if (rolled || winner) return;
    setRolling(true);
    let count = 0;
    const interval = setInterval(() => {
      setDice(Math.ceil(Math.random()*6));
      count++;
      if (count > 8) {
        clearInterval(interval);
        const val = Math.ceil(Math.random()*6);
        setDice(val);
        setRolling(false);
        setRolled(true);
        // Check if any move possible
        const ps = pieces[currentColor];
        const canMove = ps.some(p => canPieceMove(p, val, pieces));
        if (!canMove) {
          setMsg(`${currentColor.toUpperCase()} rolled ${val} — No moves! Skipping.`);
          setTimeout(() => {
            setRolled(false);
            setDice(null);
            setTurn(t => (t+1)%4);
            setMsg(`${COLORS[(turn+1)%4].toUpperCase()}'s turn — Roll the dice!`);
          }, 1200);
        } else {
          setMsg(`${currentColor.toUpperCase()} rolled ${val} — Select a piece!`);
        }
      }
    }, 80);
  };

  function canPieceMove(p, d, allPieces) {
    if (p.state === "home") return d === 6;
    if (p.state === "done") return false;
    if (p.state === "path") {
      const newIdx = p.pathIdx + d;
      if (newIdx < 52) return true;
      const overshoot = newIdx - 52;
      return overshoot <= HOME_PATH[p.color].length;
    }
    if (p.state === "homepath") {
      const newIdx = p.homeIdx + d;
      return newIdx <= HOME_PATH[p.color].length;
    }
    return false;
  }

  function movePiece(pieceId) {
    if (!rolled || winner) return;
    const p = pieces[currentColor].find(x => x.id === pieceId);
    if (!p || !canPieceMove(p, dice, pieces)) return;

    let newPieces = JSON.parse(JSON.stringify(pieces));
    let piece = newPieces[currentColor].find(x => x.id === pieceId);
    let captured = false;
    let bonusTurn = dice === 6;

    if (piece.state === "home" && dice === 6) {
      const si = START_IDX[currentColor];
      piece.state = "path";
      piece.pathIdx = si;
      piece.row = PATH[si][0];
      piece.col = PATH[si][1];
    } else if (piece.state === "path") {
      const newIdx = piece.pathIdx + dice;
      if (newIdx >= 52) {
        const hIdx = newIdx - 52;
        if (hIdx < HOME_PATH[currentColor].length) {
          piece.state = "homepath";
          piece.homeIdx = hIdx;
          piece.row = HOME_PATH[currentColor][hIdx][0];
          piece.col = HOME_PATH[currentColor][hIdx][1];
        } else if (hIdx === HOME_PATH[currentColor].length) {
          piece.state = "done";
          piece.row = CENTER[0];
          piece.col = CENTER[1];
          bonusTurn = true;
        }
      } else {
        piece.pathIdx = newIdx;
        piece.row = PATH[newIdx][0];
        piece.col = PATH[newIdx][1];
        // Capture
        COLORS.forEach(c => {
          if (c === currentColor) return;
          newPieces[c] = newPieces[c].map(op => {
            if (op.state === "path" && op.row === piece.row && op.col === piece.col) {
              captured = true;
              bonusTurn = true;
              const hp = PIECE_HOME[c][op.id];
              return {...op, state:"home", pathIdx:-1, row:hp[0], col:hp[1]};
            }
            return op;
          });
        });
      }
    } else if (piece.state === "homepath") {
      const newIdx = piece.homeIdx + dice;
      if (newIdx < HOME_PATH[currentColor].length) {
        piece.homeIdx = newIdx;
        piece.row = HOME_PATH[currentColor][newIdx][0];
        piece.col = HOME_PATH[currentColor][newIdx][1];
      } else if (newIdx === HOME_PATH[currentColor].length) {
        piece.state = "done";
        piece.row = CENTER[0];
        piece.col = CENTER[1];
        bonusTurn = true;
      }
    }

    setPieces(newPieces);
    setSelected(null);

    // Check win
    if (newPieces[currentColor].every(x => x.state === "done")) {
      setWinner(currentColor);
      setMsg(`🎉 ${currentColor.toUpperCase()} WINS!`);
      setRolled(false);
      return;
    }

    if (bonusTurn) {
      setRolled(false);
      setDice(null);
      setMsg(`${currentColor.toUpperCase()} gets another turn!${captured?" (Captured!)" : ""}`);
    } else {
      const next = (turn+1)%4;
      setRolled(false);
      setDice(null);
      setTurn(next);
      setMsg(`${COLORS[next].toUpperCase()}'s turn — Roll the dice!`);
    }
  }

  // Build cell map
  const cellMap = {};
  COLORS.forEach(color => {
    pieces[color].forEach(p => {
      if (p.state === "done") return;
      const k = posKey(p.row, p.col);
      if (!cellMap[k]) cellMap[k] = [];
      cellMap[k].push(p);
    });
  });

  const CELL = 42;
  const GRID = 15;
  const SIZE = CELL * GRID;

  function getCellColor(r, c) {
    // Home zones
    if (r<=3 && c<=3) return COLOR_LIGHT.red;
    if (r<=3 && c>=11) return COLOR_LIGHT.blue;
    if (r>=11 && c>=11) return COLOR_LIGHT.green;
    if (r>=11 && c<=3) return COLOR_LIGHT.yellow;
    // Home paths
    if (r===7 && c>=1 && c<=5) return COLOR_LIGHT.red;
    if (c===7 && r>=1 && r<=5) return COLOR_LIGHT.blue;
    if (r===7 && c>=9 && c<=13) return COLOR_LIGHT.green;
    if (c===7 && r>=9 && r<=13) return COLOR_LIGHT.yellow;
    // Center
    if (r>=6 && r<=8 && c>=6 && c<=8) return "#ecf0f1";
    // Safe squares (star positions)
    const safe = [[6,2],[2,8],[8,12],[12,6],[6,1],[1,8],[8,13],[13,6]];
    if (safe.some(([sr,sc])=>sr===r&&sc===c)) return "#f8f8b0";
    return "#fff";
  }

  function isSelectable(p) {
    return rolled && p.color === currentColor && canPieceMove(p, dice, pieces);
  }

  const doneCounts = {};
  COLORS.forEach(c => { doneCounts[c] = pieces[c].filter(p=>p.state==="done").length; });

  return (
    <div style={{minHeight:"100vh",background:"#1a1a2e",display:"flex",flexDirection:"column",alignItems:"center",padding:"16px",fontFamily:"sans-serif"}}>
      <h1 style={{color:"#fff",margin:"0 0 8px",fontSize:"28px",letterSpacing:"2px"}}>🎲 LUDO</h1>

      {/* Scoreboard */}
      <div style={{display:"flex",gap:"12px",marginBottom:"10px"}}>
        {COLORS.map(c=>(
          <div key={c} style={{background:COLOR_HEX[c],borderRadius:"8px",padding:"4px 12px",color:"#fff",fontWeight:"bold",fontSize:"13px",opacity:c===currentColor?1:0.6,border:c===currentColor?"2px solid #fff":"2px solid transparent"}}>
            {c[0].toUpperCase()}: {doneCounts[c]}/4
          </div>
        ))}
      </div>

      {/* Message */}
      <div style={{background:"#16213e",color:"#eee",borderRadius:"10px",padding:"8px 20px",marginBottom:"10px",fontSize:"15px",fontWeight:"bold",textAlign:"center"}}>
        {msg}
      </div>

      {/* Board */}
      <div style={{position:"relative",width:SIZE,height:SIZE,border:"3px solid #333",borderRadius:"4px",background:"#fff",flexShrink:0}}>
        {/* Grid lines */}
        {Array.from({length:GRID}).map((_,r)=>
          Array.from({length:GRID}).map((_,c)=>{
            const bg = getCellColor(r,c);
            return (
              <div key={`${r},${c}`} style={{
                position:"absolute",left:c*CELL,top:r*CELL,
                width:CELL,height:CELL,
                background:bg,
                border:"0.5px solid #ccc",
                boxSizing:"border-box",
              }}/>
            );
          })
        )}

        {/* Center triangle decorations */}
        {[
          {color:"red",points:`${6*CELL},${6*CELL} ${9*CELL},${6*CELL} ${7.5*CELL},${7.5*CELL}`},
          {color:"blue",points:`${9*CELL},${6*CELL} ${9*CELL},${9*CELL} ${7.5*CELL},${7.5*CELL}`},
          {color:"green",points:`${6*CELL},${9*CELL} ${9*CELL},${9*CELL} ${7.5*CELL},${7.5*CELL}`},
          {color:"yellow",points:`${6*CELL},${6*CELL} ${6*CELL},${9*CELL} ${7.5*CELL},${7.5*CELL}`},
        ].map(({color,points})=>(
          <svg key={color} style={{position:"absolute",top:0,left:0,width:SIZE,height:SIZE,pointerEvents:"none"}} viewBox={`0 0 ${SIZE} ${SIZE}`}>
            <polygon points={points} fill={COLOR_HEX[color]} opacity="0.7"/>
          </svg>
        ))}

        {/* Home base circles */}
        {COLORS.map(color => {
          const [[r1,c1]] = [HOME_COLS[color]];
          const cx = (c1+1)*CELL; const cy = (r1+1)*CELL;
          return (
            <svg key={color} style={{position:"absolute",top:0,left:0,width:SIZE,height:SIZE,pointerEvents:"none"}} viewBox={`0 0 ${SIZE} ${SIZE}`}>
              <circle cx={cx} cy={cy} r={CELL*1.2} fill={COLOR_HEX[color]} opacity="0.25"/>
            </svg>
          );
        })}

        {/* Pieces */}
        {COLORS.flatMap(color =>
          pieces[color].filter(p=>p.state!=="done").map(p => {
            const sel = isSelectable(p);
            const isSelected = selected === p.id && p.color === currentColor;
            const stack = cellMap[posKey(p.row,p.col)] || [];
            const idx = stack.findIndex(x=>x.id===p.id&&x.color===p.color);
            const offX = stack.length>1 ? (idx%2)*10-5 : 0;
            const offY = stack.length>1 ? Math.floor(idx/2)*10-5 : 0;
            return (
              <div key={`${color}-${p.id}`}
                onClick={()=>{ if(sel){setSelected(p.id); movePiece(p.id);} }}
                style={{
                  position:"absolute",
                  left: p.col*CELL + CELL/2 - 12 + offX,
                  top: p.row*CELL + CELL/2 - 12 + offY,
                  width:24, height:24,
                  borderRadius:"50%",
                  background: COLOR_HEX[color],
                  border: `3px solid ${isSelected?"#fff":COLOR_DARK[color]}`,
                  boxShadow: sel ? `0 0 0 3px #fff, 0 0 10px ${COLOR_HEX[color]}` : "0 2px 4px rgba(0,0,0,0.3)",
                  cursor: sel ? "pointer" : "default",
                  display:"flex",alignItems:"center",justifyContent:"center",
                  color:"#fff",fontWeight:"bold",fontSize:"11px",
                  zIndex: sel?10:5,
                  transform: sel ? "scale(1.15)" : "scale(1)",
                  transition:"transform 0.15s, box-shadow 0.15s",
                }}
              >
                {p.id+1}
              </div>
            );
          })
        )}

        {/* Center star */}
        <div style={{position:"absolute",left:7*CELL,top:7*CELL,width:CELL,height:CELL,display:"flex",alignItems:"center",justifyContent:"center",fontSize:"22px",zIndex:20}}>⭐</div>
      </div>

      {/* Controls */}
      <div style={{display:"flex",gap:"16px",marginTop:"14px",alignItems:"center"}}>
        <div style={{
          background: rolling?"#7f8c8d":"#2ecc71",
          color:"#fff",borderRadius:"12px",padding:"12px 32px",
          fontSize:"18px",fontWeight:"bold",cursor:rolled||winner||rolling?"default":"pointer",
          opacity:rolled||winner?0.5:1,
          boxShadow:"0 4px 15px rgba(46,204,113,0.4)",
          transition:"background 0.2s",
          userSelect:"none",
        }} onClick={rollDice}>
          {rolling ? "🎲 ..." : `🎲 Roll (${currentColor[0].toUpperCase()})`}
        </div>
        {dice && !rolling && (
          <div style={{background:"#fff",borderRadius:"10px",width:"52px",height:"52px",display:"flex",alignItems:"center",justifyContent:"center",fontSize:"28px",fontWeight:"bold",color:COLOR_HEX[currentColor],boxShadow:"0 4px 12px rgba(0,0,0,0.3)",border:`3px solid ${COLOR_HEX[currentColor]}`}}>
            {["","1️⃣","2️⃣","3️⃣","4️⃣","5️⃣","6️⃣"][dice]}
          </div>
        )}
      </div>

      {winner && (
        <div style={{marginTop:"16px",background:COLOR_HEX[winner],color:"#fff",borderRadius:"16px",padding:"16px 32px",fontSize:"22px",fontWeight:"bold",textAlign:"center",boxShadow:`0 8px 30px ${COLOR_HEX[winner]}88`}}>
          🏆 {winner.toUpperCase()} WINS! 🎉
          <div style={{marginTop:"10px"}}>
            <span style={{background:"rgba(0,0,0,0.2)",borderRadius:"8px",padding:"6px 16px",cursor:"pointer",fontSize:"15px"}}
              onClick={()=>{setPieces(initPieces());setTurn(0);setDice(null);setRolled(false);setSelected(null);setWinner(null);setMsg("Red's turn — Roll the dice!");}}>
              🔄 Play Again
            </span>
          </div>
        </div>
      )}

      <div style={{color:"#666",fontSize:"12px",marginTop:"10px",textAlign:"center"}}>
        Roll 6 to bring a piece out • Glowing pieces = clickable moves • Capture opponents for bonus turn
      </div>
    </div>
  );
}
