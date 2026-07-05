% =============================================================================
% SELF-BALANCING ROBOT - Cyberpunk Real-Time MATLAB Dashboard
% MATLAB R2025a compatible
% Serial data format:
% Angle,AngleDev,Integral,PidOut,Gyro,isFallen
% =============================================================================

function SelfBalancingRobotCyberpunkDashboard()

% ---------------- CONFIGURATION ----------------
COM_PORT   = 'COM10';      % Same COM port variable
BAUD_RATE  = 115200;       % Same baud rate
BUFFER_LEN = 240;
FALL_ANGLE = 35.0;

% ---------------- NEON PALETTE ----------------
CLR_VOID    = [0.000 0.000 0.010];
CLR_PANEL   = [0.015 0.025 0.045];
CLR_PANEL2  = [0.020 0.040 0.070];
CLR_GRID    = [0.060 0.120 0.160];
CLR_CYAN    = [0.000 0.950 1.000];
CLR_MAGENTA = [1.000 0.050 0.650];
CLR_LIME    = [0.300 1.000 0.150];
CLR_AMBER   = [1.000 0.720 0.050];
CLR_RED     = [1.000 0.100 0.120];
CLR_WHITE   = [0.900 0.960 1.000];
CLR_DIM     = [0.300 0.430 0.520];

% ---------------- DATA BUFFERS ----------------
angleBuf    = zeros(1, BUFFER_LEN);
angleDevBuf = zeros(1, BUFFER_LEN);
integralBuf = zeros(1, BUFFER_LEN);
pidBuf      = zeros(1, BUFFER_LEN);
gyroBuf     = zeros(1, BUFFER_LEN);
fallenBuf   = zeros(1, BUFFER_LEN);
timeBuf     = 1:BUFFER_LEN;

maxTilt = 0;
fallCount = 0;
lastFallen = 0;
terminalLines = repmat({''}, 10, 1);
lastLogTime = -inf;

% ---------------- BUILD FIGURE FIRST ----------------
fig = figure( ...
    'Name', 'CYBERPUNK SELF-BALANCING ROBOT TERMINAL', ...
    'Color', CLR_VOID, ...
    'Units', 'normalized', ...
    'OuterPosition', [0.01 0.02 0.98 0.96], ...
    'NumberTitle', 'off', ...
    'MenuBar', 'none', ...
    'ToolBar', 'none');

annotation(fig, 'textbox', [0 0.935 1 0.065], ...
    'String', 'CYBERDECK // SELF-BALANCING ROBOT // REAL-TIME NEURAL TELEMETRY', ...
    'Color', CLR_CYAN, ...
    'FontName', 'Consolas', ...
    'FontSize', 15, ...
    'FontWeight', 'bold', ...
    'HorizontalAlignment', 'center', ...
    'VerticalAlignment', 'middle', ...
    'EdgeColor', CLR_MAGENTA, ...
    'LineWidth', 1.2, ...
    'BackgroundColor', [0.000 0.010 0.020]);

% Layout
mL = 0.035; mR = 0.025; mB = 0.055; mT = 0.100;
gapX = 0.020; gapY = 0.030;
cW = (1 - mL - mR - 2 * gapX) / 3;
rH = (1 - mT - mB - 2 * gapY) / 3;

c1 = mL;
c2 = c1 + cW + gapX;
c3 = c2 + cW + gapX;

r3 = mB;
r2 = r3 + rH + gapY;
r1 = r2 + rH + gapY;

% ---------------- AXES HELPER ----------------
    function ax = cyberAx(pos, ttl)
        ax = axes('Parent', fig, 'Position', pos, ...
            'Color', CLR_PANEL, ...
            'XColor', CLR_DIM, ...
            'YColor', CLR_DIM, ...
            'GridColor', CLR_GRID, ...
            'MinorGridColor', CLR_GRID * 0.7, ...
            'XGrid', 'on', ...
            'YGrid', 'on', ...
            'XMinorGrid', 'on', ...
            'YMinorGrid', 'on', ...
            'GridAlpha', 0.75, ...
            'MinorGridAlpha', 0.35, ...
            'FontName', 'Consolas', ...
            'FontSize', 8, ...
            'TickDir', 'out', ...
            'Box', 'on', ...
            'LineWidth', 1.0);
        title(ax, ttl, ...
            'Color', CLR_CYAN, ...
            'FontName', 'Consolas', ...
            'FontSize', 9, ...
            'FontWeight', 'bold');
    end

% ---------------- PANEL 1: WIREFRAME ROBOT ----------------
axRobot = cyberAx([c1 r1 cW rH], 'WIREFRAME ROBOT BLUEPRINT');
hold(axRobot, 'on');
axis(axRobot, 'equal');
axis(axRobot, [-1.6 1.6 -1.8 1.8]);
set(axRobot, 'XTick', [], 'YTick', []);

plot(axRobot, [-1.6 1.6], [-1.28 -1.28], 'Color', CLR_GRID, 'LineWidth', 1.2);
plot(axRobot, [-1.4 1.4], [-1.40 -1.40], 'Color', CLR_GRID * 0.8, 'LineStyle', ':');

bodyX = [-0.22 0.22 0.22 -0.22 -0.22];
bodyY = [-0.65 -0.65 0.58 0.58 -0.65];
headX = [-0.16 0.16 0.16 -0.16 -0.16];
headY = [0.66 0.66 0.92 0.92 0.66];

hBodyGlow = plot(axRobot, bodyX, bodyY, 'Color', CLR_CYAN * 0.45, 'LineWidth', 6);
hBody     = plot(axRobot, bodyX, bodyY, 'Color', CLR_CYAN, 'LineWidth', 1.5);
hHeadGlow = plot(axRobot, headX, headY, 'Color', CLR_MAGENTA * 0.45, 'LineWidth', 5);
hHead     = plot(axRobot, headX, headY, 'Color', CLR_MAGENTA, 'LineWidth', 1.3);

hSpine = plot(axRobot, [0 0], [-0.65 0.92], 'Color', CLR_LIME, 'LineWidth', 1.0);
hEyeL = plot(axRobot, -0.07, 0.79, 'o', 'MarkerSize', 6, 'MarkerFaceColor', CLR_CYAN, 'MarkerEdgeColor', 'none');
hEyeR = plot(axRobot,  0.07, 0.79, 'o', 'MarkerSize', 6, 'MarkerFaceColor', CLR_CYAN, 'MarkerEdgeColor', 'none');

wheelTheta = linspace(0, 2*pi, 48);
wheelLX = -0.24 + 0.11 * cos(wheelTheta);
wheelLY = -0.72 + 0.11 * sin(wheelTheta);
wheelRX =  0.24 + 0.11 * cos(wheelTheta);
wheelRY = -0.72 + 0.11 * sin(wheelTheta);

hWheelL = plot(axRobot, wheelLX, wheelLY, 'Color', CLR_AMBER, 'LineWidth', 1.2);
hWheelR = plot(axRobot, wheelRX, wheelRY, 'Color', CLR_AMBER, 'LineWidth', 1.2);

hExhaust1 = quiver(axRobot, -0.15, -0.85, 0, -0.10, 0, 'Color', CLR_MAGENTA, 'LineWidth', 1.4, 'MaxHeadSize', 1.8);
hExhaust2 = quiver(axRobot,  0.15, -0.85, 0, -0.10, 0, 'Color', CLR_MAGENTA, 'LineWidth', 1.4, 'MaxHeadSize', 1.8);

hAngleText = text(axRobot, 0, -1.58, 'PITCH  +0.00 deg', ...
    'Color', CLR_CYAN, ...
    'FontName', 'Consolas', ...
    'FontSize', 12, ...
    'FontWeight', 'bold', ...
    'HorizontalAlignment', 'center');

hFallText = text(axRobot, 0, 0.10, '', ...
    'Color', CLR_RED, ...
    'FontName', 'Consolas', ...
    'FontSize', 16, ...
    'FontWeight', 'bold', ...
    'HorizontalAlignment', 'center');

% ---------------- PANEL 2: ANGLE HISTORY ----------------
axAngle = cyberAx([c2 r1 cW rH], 'TILT ANGLE TRACE // DEG');
hold(axAngle, 'on');
ylim(axAngle, [-45 45]);
xlim(axAngle, [1 BUFFER_LEN]);

plot(axAngle, [1 BUFFER_LEN], [ FALL_ANGLE  FALL_ANGLE], '--', 'Color', CLR_RED, 'LineWidth', 1.1);
plot(axAngle, [1 BUFFER_LEN], [-FALL_ANGLE -FALL_ANGLE], '--', 'Color', CLR_RED, 'LineWidth', 1.1);
plot(axAngle, [1 BUFFER_LEN], [0 0], '-', 'Color', CLR_GRID, 'LineWidth', 1.0);

hAngleGlow = plot(axAngle, timeBuf, angleBuf, 'Color', CLR_CYAN * 0.45, 'LineWidth', 5);
hAngleLine = plot(axAngle, timeBuf, angleBuf, 'Color', CLR_CYAN, 'LineWidth', 1.6);
ylabel(axAngle, 'deg', 'Color', CLR_DIM);

% ---------------- PANEL 3: RADAR TILT GAUGE ----------------
axGauge = cyberAx([c3 r1 cW rH], 'POLAR RADAR TILT GAUGE');
hold(axGauge, 'on');
axis(axGauge, 'equal');
axis(axGauge, [-1.25 1.25 -0.25 1.25]);
set(axGauge, 'XTick', [], 'YTick', []);

for rr = 0.25:0.25:1.0
    th = linspace(0, pi, 120);
    plot(axGauge, rr*cos(th), rr*sin(th), 'Color', CLR_GRID, 'LineWidth', 0.8);
end

for deg = -40:10:40
    th = deg2rad(90 - deg);
    plot(axGauge, [0 cos(th)], [0 sin(th)], 'Color', CLR_GRID * 0.9, 'LineWidth', 0.7);
    text(axGauge, 1.08*cos(th), 1.08*sin(th), sprintf('%+d', deg), ...
        'Color', CLR_DIM, ...
        'FontName', 'Consolas', ...
        'FontSize', 7, ...
        'HorizontalAlignment', 'center');
end

hGaugeNeedleGlow = plot(axGauge, [0 0], [0 0.9], 'Color', CLR_MAGENTA * 0.45, 'LineWidth', 8);
hGaugeNeedle = plot(axGauge, [0 0], [0 0.9], 'Color', CLR_MAGENTA, 'LineWidth', 2.0);
hGaugeDot = plot(axGauge, 0, 0, 'o', 'MarkerSize', 9, 'MarkerFaceColor', CLR_AMBER, 'MarkerEdgeColor', 'none');

hGaugeText = text(axGauge, 0, -0.13, '+0.00 deg', ...
    'Color', CLR_WHITE, ...
    'FontName', 'Consolas', ...
    'FontSize', 13, ...
    'FontWeight', 'bold', ...
    'HorizontalAlignment', 'center');

% ---------------- PANEL 4: PID OUTPUT ----------------
axPid = cyberAx([c1 r2 cW rH], 'MOTOR COMMAND // PID OUTPUT');
hold(axPid, 'on');
ylim(axPid, [-2200 2200]);
xlim(axPid, [1 BUFFER_LEN]);

plot(axPid, [1 BUFFER_LEN], [ 2000  2000], '--', 'Color', CLR_RED * 0.85, 'LineWidth', 0.9);
plot(axPid, [1 BUFFER_LEN], [-2000 -2000], '--', 'Color', CLR_RED * 0.85, 'LineWidth', 0.9);
plot(axPid, [1 BUFFER_LEN], [0 0], '-', 'Color', CLR_GRID, 'LineWidth', 1.0);

hPidArea = area(axPid, timeBuf, pidBuf, 'FaceColor', CLR_AMBER, 'FaceAlpha', 0.13, 'EdgeColor', 'none');
hPidLine = plot(axPid, timeBuf, pidBuf, 'Color', CLR_AMBER, 'LineWidth', 1.4);

% ---------------- PANEL 5: GYRO RATE ----------------
axGyro = cyberAx([c2 r2 cW rH], 'GYRO RATE // DEG PER SEC');
hold(axGyro, 'on');
xlim(axGyro, [1 BUFFER_LEN]);
plot(axGyro, [1 BUFFER_LEN], [0 0], '-', 'Color', CLR_GRID, 'LineWidth', 1.0);
hGyroLine = plot(axGyro, timeBuf, gyroBuf, 'Color', CLR_MAGENTA, 'LineWidth', 1.4);

% ---------------- PANEL 6: DIAGNOSTICS MATRIX ----------------
annotation(fig, 'rectangle', [c3 r2 cW rH], ...
    'Color', CLR_CYAN, ...
    'LineWidth', 1.0, ...
    'FaceColor', CLR_PANEL);

hDiagTitle = annotation(fig, 'textbox', [c3 + 0.010 r2 + rH - 0.040 cW - 0.020 0.030], ...
    'String', 'SYSTEM DIAGNOSTICS MATRIX', ...
    'Color', CLR_LIME, ...
    'FontName', 'Consolas', ...
    'FontSize', 10, ...
    'FontWeight', 'bold', ...
    'EdgeColor', 'none', ...
    'HorizontalAlignment', 'center');

hDiagText = annotation(fig, 'textbox', [c3 + 0.020 r2 + 0.020 cW - 0.040 rH - 0.065], ...
    'String', '', ...
    'Color', CLR_WHITE, ...
    'FontName', 'Consolas', ...
    'FontSize', 9, ...
    'EdgeColor', 'none', ...
    'VerticalAlignment', 'top');

% ---------------- PANEL 7: PHASE PORTRAIT ----------------
axPhase = cyberAx([c1 r3 cW rH], 'PHASE PORTRAIT // ERROR VS RATE');
hold(axPhase, 'on');
xlim(axPhase, [-45 45]);
ylim(axPhase, [-350 350]);
xlabel(axPhase, 'AngleDev', 'Color', CLR_DIM);
ylabel(axPhase, 'Gyro', 'Color', CLR_DIM);
plot(axPhase, [0 0], [-350 350], '-', 'Color', CLR_GRID, 'LineWidth', 1.0);
plot(axPhase, [-45 45], [0 0], '-', 'Color', CLR_GRID, 'LineWidth', 1.0);
hPhaseTrail = plot(axPhase, 0, 0, 'Color', CLR_CYAN, 'LineWidth', 1.2);
hPhaseDot = plot(axPhase, 0, 0, 'o', 'MarkerSize', 8, 'MarkerFaceColor', CLR_AMBER, 'MarkerEdgeColor', 'none');

% ---------------- PANEL 8: LIVE TERMINAL STREAM ----------------
annotation(fig, 'rectangle', [c2 r3 cW rH], ...
    'Color', CLR_LIME, ...
    'LineWidth', 1.0, ...
    'FaceColor', [0.000 0.018 0.012]);

hTerminalTitle = annotation(fig, 'textbox', [c2 + 0.010 r3 + rH - 0.040 cW - 0.020 0.030], ...
    'String', 'LIVE TERMINAL STREAM', ...
    'Color', CLR_LIME, ...
    'FontName', 'Consolas', ...
    'FontSize', 10, ...
    'FontWeight', 'bold', ...
    'EdgeColor', 'none', ...
    'HorizontalAlignment', 'center');

hTerminal = annotation(fig, 'textbox', [c2 + 0.018 r3 + 0.015 cW - 0.036 rH - 0.060], ...
    'String', '', ...
    'Color', CLR_LIME, ...
    'FontName', 'Consolas', ...
    'FontSize', 8.5, ...
    'EdgeColor', 'none', ...
    'VerticalAlignment', 'top');

% ---------------- PANEL 9: MOTOR SPEED GAUGE ----------------
axSpeed = cyberAx([c3 r3 cW rH], 'MOTOR SPEED VECTOR // STEPS PER SEC');
hold(axSpeed, 'on');
xlim(axSpeed, [-2200 2200]);
ylim(axSpeed, [0 1.6]);
set(axSpeed, 'YTick', [], 'XTick', [-2000 -1000 0 1000 2000]);

fill(axSpeed, [-2000 2000 2000 -2000], [0.45 0.45 0.95 0.95], CLR_PANEL2, 'EdgeColor', CLR_GRID);
fill(axSpeed, [-2000 -1100 -1100 -2000], [0.45 0.45 0.95 0.95], CLR_RED * 0.35, 'EdgeColor', 'none');
fill(axSpeed, [1100 2000 2000 1100], [0.45 0.45 0.95 0.95], CLR_RED * 0.35, 'EdgeColor', 'none');
plot(axSpeed, [0 0], [0.35 1.05], 'Color', CLR_WHITE, 'LineWidth', 1.5);

hSpeedBar = fill(axSpeed, [0 0 0 0], [0.50 0.50 0.90 0.90], CLR_CYAN, 'EdgeColor', 'none');
hSpeedText = text(axSpeed, 0, 1.30, '+0 steps/sec', ...
    'Color', CLR_WHITE, ...
    'FontName', 'Consolas', ...
    'FontSize', 11, ...
    'FontWeight', 'bold', ...
    'HorizontalAlignment', 'center');

hDirText = text(axSpeed, 0, 0.20, 'AWAITING SIGNAL', ...
    'Color', CLR_DIM, ...
    'FontName', 'Consolas', ...
    'FontSize', 9, ...
    'HorizontalAlignment', 'center');

% Footer
hFooter = annotation(fig, 'textbox', [0 0 1 0.038], ...
    'String', sprintf('CONNECTING TO %s ... CLOSE WINDOW TO STOP', COM_PORT), ...
    'Color', CLR_DIM, ...
    'FontName', 'Consolas', ...
    'FontSize', 8, ...
    'EdgeColor', 'none', ...
    'HorizontalAlignment', 'center', ...
    'VerticalAlignment', 'middle', ...
    'BackgroundColor', [0.000 0.010 0.020]);

drawnow;    % Figure opens before serial connection

% ---------------- NOW CONNECT SERIAL ----------------
s = [];
try
    s = serialport(COM_PORT, BAUD_RATE, 'Timeout', 5);
    configureTerminator(s, "LF");
    flush(s);

    set(hFooter, 'String', sprintf('CONNECTED TO %s // WAITING FOR TELEMETRY STREAM', COM_PORT), ...
        'Color', CLR_LIME);
    pushLog('SERIAL LINK ESTABLISHED');
    drawnow;
catch ME
    set(hFooter, 'String', sprintf('PORT ERROR: COULD NOT OPEN %s // %s', COM_PORT, ME.message), ...
        'Color', CLR_RED);
    pushLog('SERIAL LINK FAILURE');
    drawnow;
    return;
end

startTime = tic;
loopClock = tic;

% ---------------- MAIN LOOP ----------------
while ishandle(fig)
    loopDt = toc(loopClock);
    loopClock = tic;
    loopHz = 1 / max(loopDt, eps);

    try
        line = readline(s);
    catch
        if ~ishandle(fig)
            break;
        end
        pause(0.01);
        continue;
    end

    vals = sscanf(char(line), '%f,%f,%f,%f,%f,%d');
    if numel(vals) < 6
        continue;
    end

    angle    = vals(1);
    angleDev = vals(2);
    intVal   = vals(3);
    pidOut   = vals(4);
    gyro     = vals(5);
    fallen   = vals(6);

    angleBuf    = [angleBuf(2:end), angle];
    angleDevBuf = [angleDevBuf(2:end), angleDev];
    integralBuf = [integralBuf(2:end), intVal];
    pidBuf      = [pidBuf(2:end), pidOut];
    gyroBuf     = [gyroBuf(2:end), gyro];
    fallenBuf   = [fallenBuf(2:end), fallen];

    maxTilt = max(maxTilt, abs(angleDev));
    rmsError = sqrt(mean(angleDevBuf.^2));

    if fallen && ~lastFallen
        fallCount = fallCount + 1;
        pushLog('CRITICAL: ROBOT FALL DETECTED');
    end
    lastFallen = fallen;

    nowT = toc(startTime);
    if nowT - lastLogTime > 0.60
        if fallen
            pushLog('MOTOR FAULT // RECOVERY LOCKOUT');
        elseif abs(angleDev) > 18
            pushLog('WARNING: PITCH CRITICAL');
        elseif abs(angleDev) > 10
            pushLog('CAUTION: HIGH TILT ERROR');
        elseif abs(pidOut) > 1500
            pushLog('MOTOR SATURATION NEAR LIMIT');
        else
            pushLog('SYSTEM STABLE // PID NOMINAL');
        end
        lastLogTime = nowT;
    end

    % Update traces
    set(hAngleGlow, 'YData', angleBuf);
    set(hAngleLine, 'YData', angleBuf);

    set(hPidArea, 'YData', pidBuf);
    set(hPidLine, 'YData', pidBuf);

    set(hGyroLine, 'YData', gyroBuf);

    n = min(100, BUFFER_LEN);
    set(hPhaseTrail, 'XData', angleDevBuf(end-n+1:end), 'YData', gyroBuf(end-n+1:end));
    set(hPhaseDot, 'XData', angleDev, 'YData', gyro);

    % Speed gauge
    cp = max(-2000, min(2000, pidOut));
    if cp >= 0
        set(hSpeedBar, 'XData', [0 cp cp 0]);
    else
        set(hSpeedBar, 'XData', [cp 0 0 cp]);
    end

    stress = min(1, abs(cp) / 2000);
    speedColor = (1 - stress) * CLR_CYAN + stress * CLR_AMBER;
    set(hSpeedBar, 'FaceColor', speedColor);
    set(hSpeedText, 'String', sprintf('%+.0f steps/sec', pidOut));

    if pidOut > 50
        set(hDirText, 'String', 'FORWARD VECTOR', 'Color', CLR_CYAN);
    elseif pidOut < -50
        set(hDirText, 'String', 'REVERSE VECTOR', 'Color', CLR_MAGENTA);
    else
        set(hDirText, 'String', 'HOLDING POSITION', 'Color', CLR_DIM);
    end

    % Gauge
    gaugeDeg = max(-45, min(45, angleDev));
    gaugeTheta = deg2rad(90 - gaugeDeg);
    gx = 0.88 * cos(gaugeTheta);
    gy = 0.88 * sin(gaugeTheta);
    set(hGaugeNeedleGlow, 'XData', [0 gx], 'YData', [0 gy]);
    set(hGaugeNeedle, 'XData', [0 gx], 'YData', [0 gy]);
    set(hGaugeText, 'String', sprintf('%+.2f deg', angleDev));

    if abs(angleDev) > 18 || fallen
        set(hGaugeNeedle, 'Color', CLR_RED);
        set(hGaugeNeedleGlow, 'Color', CLR_RED * 0.45);
    else
        set(hGaugeNeedle, 'Color', CLR_MAGENTA);
        set(hGaugeNeedleGlow, 'Color', CLR_MAGENTA * 0.45);
    end

    % Robot visualizer
    updateWireRobot(angleDev, pidOut, logical(fallen));

    % Diagnostics
    diagString = sprintf([ ...
        'UPTIME              %8.2f s\n' ...
        'LOOP FREQUENCY      %8.2f Hz\n' ...
        'ANGLE               %+8.2f deg\n' ...
        'ANGLE ERROR         %+8.2f deg\n' ...
        'MAX TILT REACHED    %8.2f deg\n' ...
        'RMS ERROR           %8.2f deg\n' ...
        'GYRO RATE           %+8.2f deg/s\n' ...
        'INTEGRAL TERM       %+8.2f\n' ...
        'PID OUTPUT          %+8.0f\n' ...
        'FALL COUNT          %8d\n' ...
        'STATE               %s'], ...
        nowT, loopHz, angle, angleDev, maxTilt, rmsError, gyro, intVal, pidOut, fallCount, stateText(angleDev, fallen));

    set(hDiagText, 'String', diagString);

    set(hFooter, 'String', sprintf( ...
        't=%.1fs // ANGLE=%+.2f deg // PID=%+.0f // GYRO=%+.1f deg/s // LOOP=%.1f Hz // FALLS=%d', ...
        nowT, angle, pidOut, gyro, loopHz, fallCount), ...
        'Color', CLR_DIM);

    drawnow limitrate;
end

try
    delete(s);
catch
end

fprintf('Cyberpunk dashboard closed.\n');

% ---------------- NESTED HELPERS ----------------

    function updateWireRobot(angleDeg, motorCmd, isFallen)
        th = deg2rad(-angleDeg);
        R = [cos(th) -sin(th); sin(th) cos(th)];

        b = R * [bodyX; bodyY];
        h = R * [headX; headY];
        sp = R * [0 0; -0.65 0.92];
        eL = R * [-0.07; 0.79];
        eR = R * [ 0.07; 0.79];
        wL = R * [wheelLX; wheelLY];
        wR = R * [wheelRX; wheelRY];

        set(hBodyGlow, 'XData', b(1,:), 'YData', b(2,:));
        set(hBody, 'XData', b(1,:), 'YData', b(2,:));
        set(hHeadGlow, 'XData', h(1,:), 'YData', h(2,:));
        set(hHead, 'XData', h(1,:), 'YData', h(2,:));
        set(hSpine, 'XData', sp(1,:), 'YData', sp(2,:));
        set(hEyeL, 'XData', eL(1), 'YData', eL(2));
        set(hEyeR, 'XData', eR(1), 'YData', eR(2));
        set(hWheelL, 'XData', wL(1,:), 'YData', wL(2,:));
        set(hWheelR, 'XData', wR(1,:), 'YData', wR(2,:));

        exhaustScale = 0.15 + 0.55 * min(1, abs(motorCmd) / 2000);
        exhaustDir = sign(motorCmd);
        if exhaustDir == 0
            exhaustDir = 1;
        end

        set(hExhaust1, 'UData', -0.20 * exhaustDir, 'VData', -exhaustScale);
        set(hExhaust2, 'UData',  0.20 * exhaustDir, 'VData', -exhaustScale);

        if isFallen
            set(hBody, 'Color', CLR_RED);
            set(hBodyGlow, 'Color', CLR_RED * 0.45);
            set(hFallText, 'String', '*** FALLEN ***');
        else
            set(hBody, 'Color', CLR_CYAN);
            set(hBodyGlow, 'Color', CLR_CYAN * 0.45);
            set(hFallText, 'String', '');
        end

        set(hAngleText, 'String', sprintf('PITCH %+0.2f deg', angleDeg));
    end

    function pushLog(msg)
        stamp = datestr(now, 'HH:MM:SS.FFF');
        terminalLines = [{sprintf('[%s] %s', stamp, msg)}; terminalLines(1:end-1)];
        set(hTerminal, 'String', strjoin(terminalLines, newline));
    end

    function txt = stateText(angleError, isFallen)
        if isFallen
            txt = 'FALLEN';
        elseif abs(angleError) < 2
            txt = 'LOCKED/STABLE';
        elseif abs(angleError) < 10
            txt = 'BALANCING';
        elseif abs(angleError) < 18
            txt = 'HIGH CORRECTION';
        else
            txt = 'PITCH CRITICAL';
        end
    end

end