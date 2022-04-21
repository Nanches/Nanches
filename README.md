import http from 'k6/http';
import { check } from 'k6';
import { getApiHost, getUsers } from './config/index.js';
import { buildAuthHeaders, buildHeaders, getRandomItem, shuffle, isNil, random } from './utils/index.js';

const API_HOST = getApiHost();
const USERS = getUsers();
const OTP_CODE = '123456';
let lastRequest = null;

export let options = {
  vus: USERS.length,
  duration: '10s'
};

const signIn = ({ email, password }) => {
  const url = `${API_HOST}/auth/sign_in`;
  const payload = JSON.stringify({ email, password });
  const params = { headers: buildHeaders() };

  const req = http.post(url, payload, params);

  check(req, {
    'status is 200 for login request': (r) => r.status === 200,
    'returns data': (r) => !!r.json().data,
    'is correct user': (r) => {
      const data = r.json().data;
      if (!data) console.log(`Invalid password for user: ${email}`);
      return data.uid === email
    }
  });

  lastRequest = req;
};

const getBankAccounts = () => {
  const url = `${API_HOST}/v1/investor/bank_accounts`;
  const params = { headers: buildHeaders(buildAuthHeaders(lastRequest.headers)) };
  console.log(JSON.stringify(params))

  const req = http.get(url, params);

  check(req, {
    'status is 200 for bank accounts': (r) => r.status === 200,
    'returns the list of user bank accounts': (r) => {
      const { bank_accounts } = r.json();
      return !isNil(bank_accounts);
    },
  });
  console.log(JSON.stringify(req.json()))

  lastRequest = req;
  return req.json().bank_accounts;
};

const getRandomBank = (banks) => {
  return shuffle(banks || [])[0]; 
};

const withdraw = (bank) => {
  if (!bank) console.log('ayurameeee')
  const url = `${API_HOST}/v1/investor/withdraws`;
  const [amount] = shuffle(['100', '200', '1000', '350', '10', '2500'])
  const payload = JSON.stringify({
    withdraw: {
      amount,
      clabe: bank.clabe
    },
    otp_code: OTP_CODE
  });
  const params = { headers: buildHeaders(buildAuthHeaders(lastRequest.headers)) };

  const req = http.post(url, payload, params);

  check(req, {
    'status is 200 for withdraw': (r) => r.status === 200
  });

  lastRequest = req;
};

const logOut = () => { 

  const url = `${API_HOST}/auth/sign_out`;
  const params = { headers: buildHeaders(buildAuthHeaders(lastRequest.headers)) };

  const req = http.del(url, params);

  check(req, {
    'status is 200 for logout request': (r) => r.status === 200
  });

  return req;
};

export default function () {
  const user = getRandomItem(USERS);
  signIn(user);
  const banks = getBankAccounts();
  console.log('banks', banks, user);
  withdraw(getRandomBank(banks));
  logOut();
};
