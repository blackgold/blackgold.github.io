---
layout: post
title: Fast CIDR lookup
---

Say you have list of CIDR's and want to check if given IP falls in the CIDR. This is useful for making fast routing decisions. 

I have compared  SubnetVector class that does a linear lookup vs CidrMap class that uses hashmap for first level lookup and followed by binary trie.
For a list of 7000 subnets I have seen CidrMap takes about 5 micro seconds and SubnetVector takes 200 micro seconds.
For small list of CIDRs this doesnot make any difference.

{% highlight c++ %}
#include <iostream>
#include <tr1/memory>
#include <tr1/unordered_map>
#include <vector>
#include <fstream>

extern "C"
{
#include <sys/time.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
}

class node;
typedef std::tr1::shared_ptr<node> nodePtr;

class node {
 public:
    node() {
        m_leaf = false;
    }
    ~node(){}
    void setLeaf() {
       m_leaf=true;
    }
    bool isLeaf(){
       return m_leaf;
    }
    void setOne(nodePtr one) {
      m_one = one;
    }
    void setO(nodePtr o) {
      m_o = o;
    }
    nodePtr getOne() {
        return m_one;
    }
    nodePtr getO(){
        return m_o;
    }
 private:
    bool m_leaf;
    nodePtr m_one; // 1 child in bitwise tree
    nodePtr m_o; // 0 child in bitwise tree
};
  
class Trie {
 public:
      Trie(){};
      ~Trie(){};
      void addSubnet(const std::vector<char> &subnet);
      void addNode(nodePtr nd, const std::vector<char> &subnet, int position);
      bool searchSubnet(const std::vector<char> &ip);
      bool searchNode(nodePtr nd, const std::vector<char> &ip, int position);
 private:
      nodePtr m_head;
};

void Trie::addSubnet(const std::vector<char> &subnet) {
   if(m_head == NULL) {
       nodePtr nd(new node());
       m_head = nd;
   }
   addNode(m_head, subnet, 0);
}

void Trie::addNode(nodePtr nd, const std::vector<char> &subnet, int position) {
    if(position < subnet.size()) {
        if(subnet[position] == '1') {
            if(nd->getOne() == NULL) {
                nodePtr n(new node());
                nd->setOne(n);
                if(position == subnet.size()-1)
                   nd->setLeaf();
                addNode(n,subnet,position+1); 
            }
            else {
                if(position == subnet.size()-1)
                   nd->setLeaf();
                addNode(nd->getOne(),subnet,position+1);
            }
        }
        else if(subnet[position] == '0') {
            if(nd->getO() == NULL) {
                nodePtr n(new node());
                nd->setO(n);
                if(position == subnet.size()-1)
                   nd->setLeaf();
                addNode(n,subnet,position+1);
            }
            else {
                if(position == subnet.size()-1) {
                   nd->setLeaf();
                }
                addNode(nd->getO(),subnet,position+1);
            }
        }
        else {
           abort();
        }
    }
}

bool Trie::searchSubnet(const std::vector<char> &ip) {
  if(m_head == NULL || ip.size() == 0)
     return false;
  return searchNode(m_head,ip,0);
}

bool Trie::searchNode(nodePtr nd, const std::vector<char> &ip, int position) {
   if(ip[position] == '0') {
     if(nd->isLeaf()){
        return true;
     }    
     else if(nd->getO() != NULL) {
        return searchNode(nd->getO(),ip,position+1);
     }
     else {
        return false;
     }  
   }
   else if(ip[position] == '1') {
     if(nd->isLeaf()) {
        return true;
     }
     else if(nd->getOne() != NULL) {
        return searchNode(nd->getOne(),ip,position+1);
     }
     else {
        return false;
     }     
   }
   else {
       abort();
   }
}

class CidrMap {
  private:
      std::tr1::unordered_map<int,Trie> m_map; // use first 8 bits to see if the ip fallas in cidr or not
      public:
           bool put(std::string &subnet);
           bool put(in_addr_t prefix, int mask);
           bool get(std::string &ip);
           bool get(in_addr_t ip);
};

bool CidrMap::put( std::string &cidr) {
    std::string::size_type idx = cidr.find('/');
    if(idx != std::string::npos) {
        in_addr_t ip,mask;
        cidr[idx]=0;
        inet_pton(AF_INET,cidr.c_str(),&ip);
        int bits = atoi(cidr.c_str()+idx+1);
        return put(static_cast<in_addr_t>(htonl(ip)),bits);
    }
    return false;
}

bool CidrMap::put(in_addr_t prefix, int mask) {
  std::vector<char> subnet;
  int l1Index=((prefix & 0xff000000)>>24)&0x000000ff; // first eight bits used as key in hash map, value is a trie 
  prefix = prefix&0x00ffffff; // next 24 bits in trie
  prefix = prefix<<8;
  unsigned tmask = 0x800000;
  while(mask-- > 0) {
     if(prefix&tmask) {
       subnet.push_back('1');
     }
     else {
       subnet.push_back('0');
     }
     tmask=tmask>>1;
  }
  //for(int i=0;i<subnet.size();i++)
  //   std::cout << subnet[i] << " ";
  //std::cout << std::endl;
  if(m_map.find(l1Index) == m_map.end()) {
     Trie obj;
     obj.addSubnet(subnet);
     m_map[l1Index] = obj; // <<- hash the first eight bits to Trie that holds next 24 bits
  }
  else {
    m_map[l1Index].addSubnet(subnet);
  }
}

bool CidrMap::get(std::string &ip) {
  in_addr_t bip;
  inet_pton(AF_INET,ip.c_str(),&bip);
  get(static_cast<in_addr_t>(htonl(bip)));

}

bool CidrMap::get(in_addr_t ip) {
  std::vector<char> ipv;
  int l1Index=(ip & 0xff000000)>>24;
  unsigned mask = 0x80000000;
  ip = ip&0x00ffffff;
  ip = ip<<8;
  for(int i=0; i< 24; i++) {
     if(ip&mask) {
       ipv.push_back('1');
     }
     else {
       ipv.push_back('0');
     }
     mask=mask>>1;
  }
  //for(int i=0;i<ipv.size();i++)
  //   std::cout << ipv[i] << " ";
  //std::cout << std::endl;
  if(m_map.find(l1Index) == m_map.end())
     return false;
  else
     return m_map[l1Index].searchSubnet(ipv);
}


/*
 * Sample implementation for holding subnets in vector.
 * Lookup is linear in time.
 */
struct Netblock {
    in_addr_t addr;
    in_addr_t mask;
};

class SubnetVector {
 private:
   std::vector<Netblock> NetblockVec;
 public:
   void put(std::string subnet) {
       size_t idx = subnet.find('/');
       subnet[idx] = 0;
       Netblock n;
       if (inet_pton(AF_INET,subnet.c_str(),&n.addr) == 1) {
            if (inet_pton(AF_INET,subnet.c_str()+idx+1,&n.mask) == 1) {
                n.addr &= n.mask;
                NetblockVec.push_back(n);
            }
        }
   }

   bool get(std::string &ip) {
        in_addr_t bip;
        inet_pton(AF_INET,ip.c_str(),&bip);
        for (std::vector<Netblock>::iterator n = NetblockVec.begin(); n != NetblockVec.end(); ++n) {
            if (match_mask(bip,n->addr, n->mask) )
                 return true;
        }
        return false;
   }

   bool match_mask(in_addr_t remote_addr, in_addr_t addr, in_addr_t mask) {
        if((remote_addr & mask) == addr) return true;
        return false;
    }
};


/*
 * Helper functions to read cidr's/ip's  from files into vectors
 */
void fillCidr (std::vector<std::string> &cidrvec, std::string file) {
  std::ifstream infile(file.c_str());
  std::string cidr;
  while (infile >> cidr)
   cidrvec.push_back(cidr);
}

void fillip (std::vector<std::string> &ipvec, std::string file) {
  std::ifstream infile(file.c_str());
  std::string ip;
  while (infile >> ip)
   ipvec.push_back(ip);
}

int main() {
  CidrMap cmap;
  SubnetVector cvec;
  std::vector<std::string> cidrvec;
  std::vector<std::string> ipvec;

  fillCidr(cidrvec,"cidr.txt");
  fillip(ipvec,"ip.txt");

  for(std::vector<std::string>::iterator itr = cidrvec.begin(); itr != cidrvec.end(); itr++) {
      cmap.put(*itr); // fill the hashmap of tries
      cvec.put(*itr); // fill the vector
  }
   
  for(std::vector<std::string>::iterator itr = ipvec.begin(); itr != ipvec.end(); itr++) {
      struct timeval b,e;
      unsigned long long t1,t2;
      bool m,v;
      gettimeofday (&b, NULL);
      t1 = b.tv_usec + (unsigned long long)b.tv_sec * 1000000;
       m = cmap.get(*itr); // <- lookup in hashmap
      gettimeofday (&e, NULL);
      t2 = e.tv_usec + (unsigned long long)e.tv_sec * 1000000;
      std::cout << "[" << t2-t1 << "]  ";
      gettimeofday (&b, NULL);
      t1 = b.tv_usec + (unsigned long long)b.tv_sec * 1000000;
       v = cvec.get(*itr); // <- lookup in vector
      gettimeofday (&e, NULL);
      t2 = e.tv_usec + (unsigned long long)e.tv_sec * 1000000;
      std::cout << "[" << t2-t1 << "]  ";
      //std:: cout <<  m << "]  [" << v << "]  " << *itr << std::endl;
  }
} 


//sample file contains Subnet's
203.127.225.0/24
203.208.30.0/24
202.175.244.0/24
202.78.97.0/24

//sample file containing IP's
95.97.91.13
95.97.93.12

// sample output. First column is  time taken for trie lookup. second column is time taken for linear lookup. Time is in microseconds.
[6]  [222]  
[5]  [222]  
[4]  [252]  
[6]  [286]  
[4]  [218]

{% endhighlight %}
