---
layout: default
title: Portfolio |  Matched Filter
---

[Home](../../../../index.md) / [Smaller Projects](../index.md) /

# Matched Filter and BER Analysis of Signals

This MATLAB code is a small section of a uni project, where digital signals are sent through baseband modulation to two different sites, through a noisy channel. The noise applied signals are then filtered and received using a matched filter, where the BER of the theoretical and experimental curves are produced.

```matlab
plotMagnitudeLabel = "Magnitude (V)";
plotTimeLabel = "Time (S)";
plotFreqLabel = "Frequency (Hz)";
plotSpectralLabel = "Power Spectral Density (dB)";

%% Task 2.1
symbolSamples = 50;
fftSamples = 8000; % increase samples for fft, to better represent the sinc function
symbolAmplitude = 5;
Rs = ChannelDataRate;
symbolPeriod = 1 / Rs;


% compute the frequency and time vector
t = linspace(0, symbolPeriod, symbolSamples + 1); t = t(1:end-1);
ts = t(2) - t(1);
Fs = 1/ts;
f = linspace(-Fs/2, Fs/2 - Fs/fftSamples, fftSamples + 1); f = f(1:end-1);


% construct the signals using the time vector
S1 = symbolAmplitude * ones(1, symbolSamples);
SO = -S1;

S1_fft = fft(S1, fftSamples)/length(S1);
SO_fft = fft(SO, fftSamples)/length(SO);

% plot the time and frequency domain of the symbols
figure(),

lims = autoscale({S1,SO},1);

subplot(2,3,1);
plotSignal(t, S1, ...
    plotTimeLabel,plotMagnitudeLabel,'S1 Time Domain',lims);

subplot(2,3,2);
plotSignal(f, abs(fftshift(S1_fft)), ...
    plotFreqLabel,plotMagnitudeLabel,'S1 Frequency Spectrum',[]);
subplot(2,3,3);
plotSignal(f, convertToDB(abs(fftshift(S1_fft))), ...
    plotFreqLabel,plotSpectralLabel,'S1 Spectral Density',[]);
ylim([-40,10]);
xlim([-4*Rs, 4*Rs]);

subplot(2,3,4);
plotSignal(t, SO, ...
    plotTimeLabel,plotMagnitudeLabel,'SO Time Domain',lims);

subplot(2,3,5);
plotSignal(f, abs(fftshift(SO_fft)), ...
    plotFreqLabel,plotMagnitudeLabel,'SO Frequency Spectrum',[]);

subplot(2,3,6);
plotSignal(f, convertToDB(abs(fftshift(SO_fft))), ...
    plotFreqLabel,plotSpectralLabel,'SO Spectral Density',[]);
ylim([-40,10]);
xlim([-4*Rs, 4*Rs]);

ntnBandwidth = 2 / symbolPeriod; % calculate null to null bandwidth
S1Energy = sum(S1.^2)*ts; % calculate energy - perform integral of s(t)^2

% encapsulate all necessary data for a symbol, excluding the symbol itself,
% to make it easier to pass into functions since commonly multiple
% variables are required
symbolData = struct( ...
    'energy', S1Energy, ...
    'samples', symbolSamples,...
    'period', symbolPeriod,...
    'amplitude', symbolAmplitude,...
    'bandwidth', ntnBandwidth,...
    'ts', ts...
    );

clear SO_fft S1_fft; % clear high memory usage data structures that are unused in the rest of the program

fprintf('Null to null bandwidth = %f Hz, Average Energy = %f J \n', ntnBandwidth, S1Energy); % %.2f for 2 decimal places
%% Task 2.2

% convert message to binary
message = 'Clay Nimmo';
%message = 'LKMKNIOHOPKOPJNKNBK'; % different message to test changes in spectrum

messageDec = double(message); % Character to ASCII
dec2binary = de2bi(messageDec,7,"left-msb"); % Convert ASCII to 7-bit
binMessage = reshape(dec2binary.',1,[]); % create the binary message vector
% convert the 0s into -1, matching the S0 = -S1 relationship
binMessageScaled = binMessage * 2 - 1;

% encapsulate the modulation into a function, for use in later parts
function messageSignal = modulateBits(bitstream, symbol, samplesPerSymbol)
    totalLength = samplesPerSymbol * numel(bitstream);
    message = zeros(1,totalLength); % preallocate the size of message signal
    message(1:samplesPerSymbol:end) = bitstream; % stretch each bit in the bitstream by the amount of samples
    message = conv(message, symbol);
    message = message(1:totalLength);
    messageSignal = message;
end 

% map the bits into a stream of pulses
messageSignal = modulateBits(binMessageScaled, S1, symbolSamples);


t_messageSignal = (0:length(messageSignal)-1)* ts;

figure();
plotSignal(t_messageSignal, messageSignal, ...
    plotTimeLabel, plotMagnitudeLabel, ...
    "Baseband Modulated Binary Signal", ...
    autoscale({messageSignal},2))

%% Task 2.3
MessageSignal = fft(messageSignal) / length(messageSignal);
f = linspace(-Fs/2, Fs/2, length(messageSignal)); % this does not need upsampled fft, so redefine the vector
figure();
subplot(1,2,1); % plot magnitude spectrum
plotSignal(f, abs(fftshift(MessageSignal)), ...
    plotFreqLabel, plotMagnitudeLabel, ...
    "Frequency Spectrum of Modulated Signal", []);
subplot(1,2,2); % plot spectral power density (dB)
plotSignal(f, convertToDB(abs(fftshift(MessageSignal))), ...
    plotFreqLabel, "Power Spectral Density (dB)", ...
    "Spectral Density of the Modulated Signal", []);
ylim([-30,10]);
xlim([-ntnBandwidth*2, ntnBandwidth*2]);

clear MessageSignal; % message signal uses too much memory to keep in the workspace

% frequency spectrum is similar to that of S1, however the sinc rebounds
% are replaced with smaller spectrum. The null to null bandwidth is the
% same

%% Task 2.4

% struct for the data per site, containing:
% data.distance -> distance in km to this site
% data.Prx -> recieved power of the signal
% data.SNR -> SNR power of the recieved signal
% data.signal -> the noisy signal recovered from the site
% data.BER -> bit error rate of the signal if it is recieved through a
% matched filter

% function to pre initialize a struct, so a later functions do not need to
% be changed if new values are added, by calling, for example,
% site.distance=x, instead of site = struct('distance',x, ....)
function site = initializeSiteStruct()
    site = struct( ...
       'distance', 0, ...
       'Prx', 0, ...
       'SNR', 0, ...
       'ebnodB', 0, ...
       'signal', zeros(1:10), ...
       'BER', 0 ...
    );
end

dists = [3, 4.1]; % array of distances, in km, to the different sites
numSites = length(dists);
% create and initialize a cell array of structs for all sites
siteData = cell(1, numSites);
for k = 1:numSites
    siteData{k} = initializeSiteStruct();
    siteData{k}.distance = dists(k);
end

function snr = calculateSNR(ebnodB, samples)
    snr = ebnodB + convertToDB(2) - convertToDB(samples);
end

% function to get the noisy signal, SNR, and Prx per site. symbolData is
% the struct defined in part 2.1, and must contain period and samples
% fields
function sites = getNoisySignalsAtSite(messageSignal, siteData, numSites, lossPerDistance, N0, PtxdB, symbolData, t)
    figure();
    for k = 1:numSites
        L = siteData{k}.distance * lossPerDistance; % loss in db
        PrxdB = PtxdB - L; % recieved power in db
        Ebdb = convertToDB(symbolData.energy);
        
        % get energy per bit from loss
        Ebrx = Ebdb - L;
        Ebrx = convertFromDB(Ebrx);

        ebnodB = convertToDB(Ebrx/N0);
        SNR = calculateSNR(ebnodB, symbolData.samples);

        attenuation = 10^(-L/20);
        rxSignal = messageSignal * attenuation;

        noisySignal = awgn(rxSignal, SNR, 'measured');
    
        siteData{k}.Prx = PrxdB;
        siteData{k}.SNR = SNR;
        siteData{k}.ebnodB = ebnodB;
        siteData{k}.signal = noisySignal;
        
        subplot(1,numSites,k);
        plotSignal(t, noisySignal, "Time (S)", "Magnitude (V)", "Noisy Signal at Site " + intToChar(k), autoscale({noisySignal},0.005));
        fprintf('Prx = %fdBW\n', PrxdB);
        fprintf('SNR = %fdB\n', SNR);
        fprintf('L = %fdB\n', L);
    end
    sites = siteData;
end 

N0 = 10E-13;
lossPerKm = 20;

Ptx = mean(messageSignal.^2); % power in watts of the transmitted signal from 2.2
PtxdB = convertToDB(Ptx);

% calculate the values at each location
siteData = getNoisySignalsAtSite(messageSignal, siteData, numSites, lossPerKm, N0, PtxdB, symbolData, t_messageSignal);

%% Task 2.6
hopt = fliplr(S1);
tbin = t_messageSignal(symbolSamples:symbolSamples:end); % time vector to line up bits
figure();
plotSignal(t, hopt, "Time (S)", "Amplitude (V)", "", []);

% function to populate the bit error rate (BER) value of the site struct
function sites = calculateBitErrorRateFromSites(siteData, numSites)
    for k = 1:numSites
        ebno = convertFromDB(siteData{k}.ebnodB);
        bitErrorRate = qfunc(sqrt(2*ebno));
        siteData{k}.BER = bitErrorRate;
    end
    sites = siteData;
end

siteData = calculateBitErrorRateFromSites(siteData, numSites);


% the bit errors are logged as 0 for site A, and 2E-4 for site B. This is
% reasonable, since the noise at site A does not extend to a point where it
% changes sign, so for example a bit of 1 is always positive. This is not
% the case for site B, where at some points the noise crosses over the y=0
% position, which can contribute to bit errors
%% Task 2.7 + 2.8

% function to apply the matched filter, returning a struct with the output
% and the sampled symbols (for example, the message with 10 pulses will have
% length(symbols) = 10
function result = applyMatchedFilter(signal, hopt, symbolData)
    % filter is a convolution, but without the added length(hopt) samples
    mf = filter(hopt, 1, signal);
    mf = mf/(max(mf)); % normalize for cleaner plotting
    
    symbols = mf(symbolData.samples : symbolData.samples : end);
    result = struct( ...
        'matchedOutput', mf, ...
        'symbols', symbols ...
    );
end

% function to decide whether the symbol is bit 0 or 1
function symbols = decideSymbols(matchedSymbols)
    symbols = matchedSymbols >= 0;
end

cleanedSiteSignals = cell(1, numSites);
figure();
for k = 1:numSites + 1
    if k == 1
        signal = messageSignal;
        plotTitle = "Matched Filter Output of Original Signal";
    else
        signal = siteData{k-1}.signal;
        plotTitle = "Matched Filter Output at Site " + intToChar(k-1);
    end
    
    result = applyMatchedFilter(signal, hopt, symbolData);
     
    % store the value per symbol for later analysis
    if k > 1
        cleanedSiteSignals{k-1} = result.symbols;
    end
    
    % plot
    subplot(numSites+1,1, k);
    plotSignal(t_messageSignal, result.matchedOutput, "Time (S)", "Magnitude (V)", plotTitle, autoscale({result.matchedOutput},0.2));
    hold on;
    stem(tbin,result.symbols,'r') % plot sampled points
    legend('Filter output', 'Sampled points')
    hold off;
end
%% Task 2.9 - A; decode the message

function message = decodeMessage(inputMessage)
    reshapedMessage = reshape(inputMessage, 7, []);
    message = char(bi2de(reshapedMessage.','left-msb'));
end

function diff = bitDifference(originalBitStream, retrievedBitStream)
    diff = sum(originalBitStream ~= retrievedBitStream);
end

for k = 1:numSites
    cleanedSignal = cleanedSiteSignals{k};
        
    % matched filter decision: x < 0 → 0, x >= 0 → 1
    cleanedSignalDecide = decideSymbols(cleanedSignal);
    decodedMessage = decodeMessage(cleanedSignalDecide);

    disp(bitDifference(binMessage,cleanedSignalDecide)/length(cleanedSignalDecide));
    fprintf(decodedMessage);
    fprintf("\n");
end

%% Task 2.9 - B; BER curve


randomBitCount = 500000;
iterations = 4;

rndSignal = modulateBits(rndBitStream, S1, symbolSamples);
t_rnd = (0:length(rndSignal)-1)* ts;

minebno = -10; % minimum ebno to plot
maxebno = 15; % maximum ebno to plot
ebnosamples = 20; % number of points between min and max to plot

ebnoCurve = linspace(minebno,maxebno,ebnosamples);
SNRCurve = calculateSNR(ebnoCurve, symbolData.samples);
ber = zeros(size(ebnoCurve));

% preallocate variables
sig = rndSignal;
cleanSig = rndBitStream;
decide = cleanSig;

for k = 1:length(ebnoCurve)
    rng(k);
    ber(k) = 0;
    errors = zeros(1, iterations); % vectorized number of errors
    for N = 1:iterations % run in iterations, to avoid out of memory errors
        rndBitStream = randi([0,1],1,randomBitCount) * 2 - 1;
        rndSignal = modulateBits(rndBitStream, S1, symbolSamples);

        sig = awgn(rndSignal, SNRCurve(k), 'measured');

        result = applyMatchedFilter(sig, hopt, symbolData);
        decide = decideSymbols(result.symbols);

        errors(N) = bitDifference((rndBitStream + 1) / 2, decide);
    end
    ber(k) = mean(errors) / randomBitCount;

end

% Theoretical BER curve for BPSK
ebnoLinear = convertFromDB(ebnoCurve);          % convert dB to linear
berTheory = qfunc(sqrt(2*ebnoLinear));   % theoretical BPSK BER

figure;
hold on;
grid on;

plot(ebnoCurve, ber, 'LineWidth', 1.5);
plot(ebnoCurve, berTheory, '--k', 'LineWidth', 1);   % dashed black line

% add points to the graph for the sites
xPoints = zeros(1:numSites);
yPoints = xPoints;
for k = 1:numSites
    xPoints(k) = siteData{k}.ebnodB;
    yTest = interp1(ebnoCurve, ber, xPoints(k),'linear');
    yPoints(k) = max(yTest, siteData{k}.BER);
end
plot(xPoints, yPoints, 'ms','MarkerSize',3,'LineWidth',2);
% add text to the added points
for k = 1:length(xPoints)
    label = sprintf('Site %s (%.2f, %.2e)', ...
                    intToChar(k), xPoints(k), yPoints(k));
    text(xPoints(k), yPoints(k)*3,label);
end
set(gca, 'YScale', 'log');
xlabel('Eb/N0 (dB)');
ylabel('Bit Error Rate (BER)');
title('Theoretical and Experiment BER');

legend("Simulated BER","Theoretical BER");

hold off;

```