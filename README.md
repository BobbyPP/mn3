EVAL_INTERPOLATOR_C

function Nk = eval_interpolator_c(tip, eps)
  
  x = linspace(-pi, pi, 1001); %diviziunile echidistante dintre -pi si pi
  
  k = 2;
  E = 0;
  Eprev = 1000;
  P = zeros(1001, 1);
  Pprev = zeros(1001, 1);
  nrIter = 1;
  while norm(E - Eprev) >= eps && (E < Eprev || nrIter <= 5)
    interval_interpolare = linspace(-pi, pi, 2^k + 1);
    y = exp(3 * cos(interval_interpolare)) / (2 * pi * besseli(0, 3)); %valorile functiei f pentru multimea x de valori discrete
    if tip == 1
      for i = 1 : 1001
        p(i) = Lagrange(interval_interpolare, y, x(i));
      endfor
    endif
    if tip == 2
      for i = 1 : 1001
        p(i) = Newton(interval_interpolare, y, x(i));
      endfor
    endif
    if tip == 3
      for i = 1 : 1001
        p(i) = SplineLiniar(interval_interpolare, y, x(i));
      endfor
    endif
    if tip == 4
      for i = 1 : 1001
        p(i) = SplineC2Natural(interval_interpolare, y, x(i));
      endfor
    endif
    if tip == 5
      fder1 = derivative_one(interval_interpolare);
      for i = 1 : 1001
        p(i) = SplineC2Tensionat(interval_interpolare, y, fder1, x(i));
      endfor
    endif
    if tip == 6
      for i = 1 : 1001
        p(i) = Trigonometric_polynomial(y, x(i));
      endfor
    endif
    
    if k > 2
      Eprev = E;
    endif
    E = 0;
    for i = 1 : 1001
      E = E + (abs(eval_f(x(i)) - p(i)))^2;
    endfor
    E = sqrt(2 * pi / 1001 * E);
    
    k = k + 1;
    if nrIter >= 2
      dif = E - Eprev;
    endif
    nrIter ++;
  endwhile
  
  k --;
  Nk = 2^k;
  if E >= Eprev
    Nk = inf;
  endif
  %
  for i = 1 : 1001
    a(i) = eval_f(x(i));
  endfor
  figure; hold;
  plot(x, a);
  plot(x, p, 'r');
  
endfunction

function [rez] = Lagrange (x, y, val)
  
  n = length(x);
  
  for i = 1 : n
    l(i) = 1;
    for j = 1 : n
      if j != i
        l(i) = l(i) * (val - x(j)) / (x(i) - x(j));
      endif
    endfor
  endfor
  
  L = 0;
  for j = 1 : n
    L = L + y(j) * l(j);
  endfor
  
  rez = L;
  
endfunction

function [rez] = Newton (x, y, val)
  
  n = length(x);
  A = zeros(n - 1);
  
  for i = 1 : n - 1
    A(i, 1) = (y(i + 1) - y(i)) / (x(i + 1) - x(i));
  endfor
  
  for j = 2 : n - 1
    for i = 1 : n - j
      A(i, j) = (A(i + 1, j - 1) - A(i, j - 1)) / (x(i + j) - x(i));
    endfor
  endfor
  
  rez = y(1);
  
  for i = 2 : n
    prod = 1;
    for j = 2 : i
      prod = prod * (val - x(j - 1));
    endfor
    rez = rez + A(1, i - 1) * prod;
  endfor
  
endfunction

function [rez] = SplineLiniar (x, y, val)
  
  n = length(x);
  if val < x(1) || val > x(n)
    disp ( "Acest algoritm nu accepta calcularea valorilor in afara intervalului format de punctele x." );
  else
    i = 1;
    while ( x(i) < val )
      i ++;
    endwhile
    
    if x(i) == val
      rez = y(i);
      return;
    endif
    rez = y(i - 1) + (val - x(i - 1)) / (x(i) - x(i - 1)) * (y(i) - y(i - 1));
  endif
  
endfunction

function [rez] = SplineC2Natural (x, y, val)
  
  n = length(x);
  
  if val < x(1) || val > x(n)
    disp ( "Acest algoritm nu accepta calcularea valorilor in afara intervalului format de punctele x." );
    return;
  endif
  
  for i = 1 : n
    if val == x(i)
      rez = y(i);
      return;
    endif
  endfor
  
  j = 1;
  while x(j) < val
    j ++;
  endwhile
  
  if x(j) == val
    rez = x(j);
    return;
  endif
  
  for i = 1 : n - 1
    h(i) = x(i + 1) - x(i);
  endfor
  
  %%%%%
  for i = 1 : n - 2
    diag_inf(i) = h(i);
  endfor
  diag_inf(n - 1) = 0;
  
  diag_princ(1) = 1; diag_princ(n) = 1;
  for i = 2 : n - 1
    diag_princ(i) = 2 * (h(i - 1) + h(i));
  endfor
  
  diag_sup(1) = 0;
  for i = 2 : n - 1
    diag_sup(i) = h(i);
  endfor
  %%%%%
  
  for i = 1 : n
    a(i) = y(i);
  endfor
  
  d0(1) = 0; d0(n) = 0;
  for i = 2 : n - 1
    d0(i) = 3 * ( (1 / h(i)) * (a(i + 1) - a(i)) - (1 / h(i - 1)) * (a(i) - a(i - 1)) );
  endfor
  
  c = Thomas (diag_inf, diag_princ, diag_sup, d0);
  
  for i = 1 : n - 1
    b(i) = (a(i + 1) - a(i)) / h(i) - (1 / 3) * h(i) * (2 * c(i) + c(i + 1));
  endfor
  
  for i = 1 : n - 1
    d(i) = (c(i + 1) - c(i)) / (3 * h(i));
  endfor
  
  rez = a(j - 1) + b(j - 1) * (val - x(j - 1)) + c(j - 1) * (val - x(j - 1))^2 + d(j - 1) * (val - x(j - 1))^3;
  
endfunction

function [rez] = SplineC2Tensionat (x, y, xder1, val)
  
  n = length(x);
  
  if val < x(1) || val > x(n)
    disp ( "Acest algoritm nu accepta calcularea valorilor in afara intervalului format de punctele x." );
    return;
  endif
  
  for i = 1 : n
    if val == x(i)
      rez = y(i);
      return;
    endif
  endfor
  
  j = 1;
  while x(j) < val
    j ++;
  endwhile
  
  if x(j) == val
    rez = x(j);
    return;
  endif
  
  for i = 1 : n - 1
    h(i) = x(i + 1) - x(i);
  endfor
  
  %%%%%
  for i = 1 : n - 2
    diag_inf(i) = h(i);
  endfor
  diag_inf(n - 1) = 0;
  
  diag_princ(1) = 2 * h(1); diag_princ(n) = 2 * h(n - 1);
  for i = 2 : n - 1
    diag_princ(i) = 2 * (h(i - 1) + h(i));
  endfor
  
  for i = 1 : n - 1
    diag_sup(i) = h(i);
  endfor
  %%%%%
  
  for i = 1 : n
    a(i) = y(i);
  endfor
  
  d0(1) = 3 * ((a(2) - a(1)) / h(1) - xder1(1));
  d0(n) = 3 * (xder1(n) - (a(n) - a(n - 1)) / h(n - 1));
  for i = 2 : n - 1
    d0(i) = 3 * ( (1 / h(i)) * (a(i + 1) - a(i)) - (1 / h(i - 1)) * (a(i) - a(i - 1)) );
  endfor
  
  c = Thomas (diag_inf, diag_princ, diag_sup, d0);
  
  for i = 1 : n - 1
    b(i) = (a(i + 1) - a(i)) / h(i) - (1 / 3) * h(i) * (2 * c(i) + c(i + 1));
  endfor
  
  for i = 1 : n - 1
    d(i) = (c(i + 1) - c(i)) / (3 * h(i));
  endfor
  
  rez = a(j - 1) + b(j - 1) * (val - x(j - 1)) + c(j - 1) * (val - x(j - 1))^2 + d(j - 1) * (val - x(j - 1))^3;
  
endfunction

function Sm = Trigonometric_polynomial (y, val)
  
  m = floor(length(y) / 2);
  
  for j = 1 : 2 * m
    x(j) = -pi + ((j - 1) / m) * pi;
  endfor
  
  for k = 1 : m + 1
    a(k) = 0;
    for j = 1 : 2 * m
      a(k) = a(k) + y(j) * cos((k - 1) * x(j));
    endfor
    a(k) = (1 / m) * a(k);
  endfor
  
  b(1) = 0;
  for k = 2 : m
    b(k) = 0;
    for j = 1 : 2 * m
      b(k) = b(k) + y(j) * sin((k - 1) * x(j));
    endfor
    b(k) = (1 / m) * b(k);
  endfor
  
  Sm = 0;
  for k = 2 : m
    Sm = Sm + a(k) * cos((k - 1) * val) + b(k) * sin((k - 1) * val);
  endfor
  Sm = Sm + (1 / 2) * (a(1) + a(m + 1) * cos(m * val));
  
endfunction

function [der] = derivative_one_discret (nrpct)
  
  n = length(nrpct);
  
  der(1) = nrpct(2) - nrpct(1);
  der(n) = nrpct(n) - nrpct(n - 1);
  
  for i = 2 : n - 1
    if nrpct(i - 1) <= nrpct(i)
      if nrpct(i) <= nrpct(i + 1)
        der(i) = (nrpct(i + 1) - nrpct(i - 1)) / 2;
      else
        der(i) = 0;
      endif
    endif
    if nrpct(i - 1) >= nrpct(i)
      if nrpct(i) <= nrpct(i + 1)
        der(i) = 0;
      else
        der(i) = (nrpct(i + 1) - nrpct(i - 1)) / 2;
      endif
    endif
  endfor
  
endfunction

function [x] = Thomas (a, b, c, d)
  
  n = length(b);
  
  c0(1) = c(1) / b(1);
  
  for i = 2 : n - 1
    c0(i) = c(i) / (b(i) - c0(i - 1) * a(i - 1));
  endfor
  
  d0(1) = d(1) / b(1);
  
  for i = 2 : n
    d0(i) = (d(i) - d0(i - 1) * a(i - 1)) / (b(i) - c0(i - 1) * a(i - 1));
  endfor
  
  x(n) = d0(n);
  
  for i = n - 1 : -1 : 1
    x(i) = d0(i) - c0(i) * x(i + 1);
  endfor
  
endfunction
